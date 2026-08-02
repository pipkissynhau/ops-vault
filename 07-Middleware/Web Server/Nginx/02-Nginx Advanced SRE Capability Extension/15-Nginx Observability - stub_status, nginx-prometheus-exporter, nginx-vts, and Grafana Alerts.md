# 15-Nginx Observability: stub_status, nginx-prometheus-exporter, nginx-vts, and Grafana Alerts

#Nginx #Observation #Monitor #Prometheus #Grafana #Exporter #stub_status #nginx-vts #nginx-prometheus-exporter #nginx-vts-exporter #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capabilities Expansion/15-Nginx Observability: stub_status, nginx-prometheus-exporter, nginx-vts, and Grafana Alerts.md

---

## One, Document Description

This document organizes methods for building observability at the Nginx ingress layer, focusing on how Nginx integrates with Prometheus, Grafana, and log platforms.

This document highlights:

- Nginx observability objectives
- Two Prometheus integration approaches for Nginx monitoring
- Method 1: `stub_status` + `nginx/nginx-prometheus-exporter`
- Method 2: `nginx-module-vts` + `nginx-vts-exporter`
- Comparison of advantages and disadvantages of the two approaches
- Which approach provides richer Grafana panels
- `stub_status` configuration and field explanations
- `nginx-prometheus-exporter` container deployment
- `nginx-prometheus-exporter` systemd deployment
- `nginx-module-vts` compilation installation approach
- `vts` status page configuration
- `nginx-vts-exporter` deployment
- Prometheus scraping configuration
- Grafana panel design
- Blackbox Exporter HTTP probe
- HTTPS certificate expiration monitoring
- access.log / error.log log observability
- Common alert rules
- Production considerations

This document is part of the Nginx Advanced SRE Capabilities Expansion series, the 15th article.

This document's objectives:

```text
I understand. Nginx Surveillance is not the only way.

→ Available stub_status + nginx-prometheus-exporter Base control complete.

→ I understand. nginx-vts + nginx-vts-exporter Why are the indicators richer?

→ Can you tell me what the two options are? Grafana Difference on Panel

→ They can choose the right option from the production scene.

→ Configure Prometheus Capture Nginx Indicators

→ It can be designed. Nginx Grafana Baseboard

→ Configure Nginx Common alarms

→ You can combine indicators, detection, logs to complete the barrier.
```

---

## Two, Nginx Observability Objectives

As an ingress layer, Nginx primarily needs to answer the following questions:

```text
Nginx Alive?

Nginx Port listening

Nginx Is it possible to respond normally to requests?

Is the current number of connections abnormal?

Is the request surged?

Whether there's a lot of them. 4xx / 5xx

Whether there's a lot of them. 499 / 502 / 504

Whether or not to trigger limit flow 429

Backend upstream Is it unusual?

Whether a back end has a higher error rate

Some server_name Is the flow abnormal?

Whether the request takes time to rise

upstream_response_time Raise

Could the certificate expire soon?

Is the log still properly written?

Is the disk filled with logs?

Multiple Nginx Is there only one anomaly?
```

One-sentence understanding:

```text
Nginx Observability depends not only on whether the process is alive, but also on whether the access is available, whether the flow is normal, whether the error is abnormal or whether the back end is healthy.
```

---

## Three, Two Implementation Paths for Nginx Prometheus Monitoring

Nginx commonly has two paths for integrating with Prometheus.

---

## Method 1: stub_status + nginx-prometheus-exporter

Path:

```text
Nginx Internal stub_status

→ nginx/nginx-prometheus-exporter

→ Prometheus

→ Grafana

→ Alertmanager
```

Features:

```text
Deployment simple

No re-compilation required Nginx

Use Nginx Internal stub_status Module

For most ordinary people. Nginx Basic monitoring

Low maintenance costs
```

Mainly visible:

```text
Nginx Retrievable

Active connections

Accepted connections

Handled connections

Requests total

Reading

Writing

Waiting

Basis QPS
```

Limitations:

```text
Fewer indicators

I can't see the fineness. server_name Indicators

I can't see the fineness. upstream Indicators

Can not see status code distribution

I can't see every single one. upstream Request volume and response time for back-end nodes

Grafana Panel is relatively simple
```

---

## Method 2: nginx-module-vts + nginx-vts-exporter

Path:

```text
Nginx Compiler installation nginx-module-vts

→ VTS Status Page /status/format/json

→ nginx-vts-exporter

→ Prometheus

→ Grafana

→ Alertmanager
```

May also use:

```text
Nginx Compiler installation nginx-module-vts

→ /status/format/prometheus

→ Prometheus Direct Capture
```

But in many traditional practices, it still uses:

```text
/status/format/json

→ nginx-vts-exporter
```

Features:

```text
More indicators.

You can see it. server zones

You can see it. upstream Dimensions

You can see it. cache Dimensions

You can see the status code classification.

Grafana The panels are usually richer.

Closer to an access level business perspective.
```

Mainly visible:

§

Limitations:

```text
Third-party module required

Usually need to be recompiled Nginx

Upgrade Nginx Time to reconsider module compatibility

Higher risk of change in production

Higher maintenance cost stub_status Programme

Modules and exporter Long-term maintenance needs assessment
```

---

## Four, Comparison of the Two Approaches

| Comparison Item | Method 1: stub_status + nginx-prometheus-exporter | Method 2: nginx-vts + nginx-vts-exporter |
|---|---|---|
| Deployment Difficulty | Low | Medium to High |
| Whether Requires Recompiling Nginx | Usually Not Required | Usually Required |
| Metric Richness | Basic | More Rich |
| Grafana Panel Richness | Average | More Rich |
| upstream Dimension | Basically None | Better Support |
| server_name Dimension | Basically None | Better Support |
| Status Code Classification | Depends on Logs | Can Be Obtained from VTS Metrics |
| Maintenance Cost | Low | High |
| Suitable Scenario | Basic Monitoring | Requires Fine-grained Nginx Ingress Metrics |
| Production Recommendation Priority | Prioritize | Consider Only with Clear Requirements |

---

## Five, Which Approach Provides More Grafana Panels

Conclusion:

```text
Grafana Panel abundance:

nginx-vts + nginx-vts-exporter

>

stub_status + nginx-prometheus-exporter
```

Reason:

```text
stub_status Fewer indicators

Grafana Usually only the number of connections, the total number of requests,QPSI don't know.ReadingI don't know.WritingI don't know.Waiting

nginx-vts More indicators.

Grafana You can show it. serverI don't know.upstreamAdditional dimensions, state code, flow, response time, back-end node, etc.
```

Intuitive Understanding:

```text
Methodology 1 More like basic state monitoring:

Nginx Alive?

How many connections?

QPS How much?

Reading / Writing / Waiting Is it unusual?
```

```text
Methodology 2 More like access layer traffic monitoring:

Which one? server_name High traffic?

Which one? upstream Too many mistakes?

Which back end is slow?

Which one? upstream 5xx More?

What is the distribution of virtual host state codes?

How's the flow going?
```

Production Selection Recommendation:

```text
Normal Nginx Basic monitoring
→ Methodology 1:stub_status + nginx-prometheus-exporter

Need to be richer. Grafana Panel
→ Methodology 2:nginx-vts + nginx-vts-exporter

If it's just...“Look. Nginx Are you alive?”
→ I don't need to. vts

If you're going to access layer traffic management upstream Width Viewer
→ It can be evaluated. vts Programme
```

---

## Six, Recommended Selection Principles

---

## Scenario 1: Ordinary Production Environment

Recommended:

```text
stub_status + nginx-prometheus-exporter

+

blackbox_exporter HTTP Search.

+

access.log JSON Log

+

error.log Error Key Warning
```

Reason:

```text
Deployment simple

Low maintenance costs

No re-compilation required Nginx

Fits most basic monitoring claims
```

---

## Scenario 2: Need Grafana Panels with upstream Dimension

Can Consider:

```text
nginx-module-vts + nginx-vts-exporter
```

Suitable For:

```text
Multiple server_name

Multiple upstream

I need to see it. upstream Width Request Measure

I need to see it. upstream Dimension Status Code

Need to see back-end dimension indicators

Hope. Grafana The panel is richer.

I hope the access level indicators are closer to the business perspective.
```

---

## Scenario 3: Already Has a Log Platform

If Already Has:

```text
JSON access.log

Loki / ELK / OpenSearch

Promtail / Filebeat

Grafana Log queries
```

Then Even Using Method 1, You Can Complement with Logs:

```text
Status Code Distribution

URI Dimensions

upstream_addr

upstream_status

request_time

upstream_response_time

499 / 502 / 504 / 429
```

At This Point, It's Not Necessarily Required to Use VTS.

---

## Scenario 4: Not Recommended to Directly Use VTS

Not Recommended to Directly Use VTS:

```text
Nginx System package installation. Upgrade system dependency. Package

The team isn't familiar. Nginx Compile

Very small production window.

No test environment validation

No module compatibility validation

Just wanted to see. Nginx Alive?

There are log platforms available to meet status codes and upstream Analysis
```

---

## Seven, Method 1: stub_status Basics

---

## Scenario 5: What is stub_status

`stub_status` is Nginx's built-in basic status page.

It Outputs:

```text
Active connections

accepts

handled

requests

Reading

Writing

Waiting
```

Example:

```text
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

---

## Scenario 6: Check if stub_status is Supported

Check Compilation Parameters:

```bash
nginx -V 2>&1 | grep http_stub_status_module
```

If You See:

```text
--with-http_stub_status_module
```

It Indicates Support.

---

## Scenario 7: Configure stub_status

Recommend Listening Only on the Local Address to Avoid Exposing to the Public Internet.

Configuration:

```nginx
server {
    listen 127.0.0.1:8088;
    server_name localhost;

    access_log off;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

Check Configuration:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Verify:

```bash
curl http://127.0.0.1:8088/nginx_status
```

---

## Scenario 8: Allow Monitoring Machine to Access stub_status

If the Exporter is Not on the Same Machine, You Need to Allow Access.

Example:

```nginx
server {
    listen 8088;
    server_name localhost;

    access_log off;

    location /nginx_status {
        stub_status;

        allow 127.0.0.1;
        allow 10.0.0.10;
        deny all;
    }
}
```

Verify:

```bash
ss -lntp | grep ':8088'
```

Access from Monitoring Machine:

```bash
curl http://NginxNodesIP:8088/nginx_status
```

Production Recommendation:

```text
stub_status Don't expose the Internet.

Allow only local or Prometheus / exporter Monitor access
```

---

## Eight, Method 1: nginx-prometheus-exporter Container Deployment

---

## Scenario 9: Use nginx/nginx-prometheus-exporter Container

Path:

```text
Nginx stub_status

→ nginx/nginx-prometheus-exporter Containers

→ Prometheus
```

Run Container:

```bash
docker run -d \
  --name nginx-prometheus-exporter \
  --restart=always \
  --network host \
  nginx/nginx-prometheus-exporter:latest \
  --nginx.scrape-uri=http://127.0.0.1:8088/nginx_status \
  --web.listen-address=:9113
```

Explanation:

```text
--network host
→ Containers directly using host network to facilitate access 127.0.0.1:8088

--nginx.scrape-uri
→ Assign stub_status Address

--web.listen-address
→ exporter Exposure metrics Other Organiser
```

Verify Container:

```bash
docker ps | grep nginx-prometheus-exporter
```

Check Logs:

```bash
docker logs -f nginx-prometheus-exporter
```

Verify Metrics:

```bash
curl http://127.0.0.1:9113/metrics
```

View Nginx Metrics:

```bash
curl -s http://127.0.0.1:9113/metrics | grep nginx
```

---

## Scenario 10: Notes When Not Using Host Network

If Using Ordinary Bridge Network:

```bash
docker run -d \
  --name nginx-prometheus-exporter \
  --restart=always \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:latest \
  --nginx.scrape-uri=http://HostIP:8088/nginx_status \
  --web.listen-address=:9113
```

Note:

```text
Inside the container 127.0.0.1 The container itself.

Not a host.

If stub_status Listening host 127.0.0.1No direct access inside the container

Available host Network

Or else. stub_status Listen to host Intranet IP

You can also use Docker Special hostname or custom network scheme
```

---

## Nine, Method 1: nginx-prometheus-exporter systemd Deployment

---

## Scenario 11: Running exporter in binary mode

Example:

```bash
/usr/local/bin/nginx-prometheus-exporter \
  --nginx.scrape-uri=http://127.0.0.1:8088/nginx_status \
  --web.listen-address=:9113
```

Validation:

```bash
curl http://127.0.0.1:9113/metrics
```

---

## Scenario 12: Managing exporter with systemd

Create user:

```bash
useradd -r -s /sbin/nologin nginx_exporter
```

Create service:

```bash
vi /etc/systemd/system/nginx-prometheus-exporter.service
```

Content:

```ini
[Unit]
Description=Nginx Prometheus Exporter
After=network.target nginx.service

[Service]
User=nginx_exporter
Group=nginx_exporter
Type=simple
ExecStart=/usr/local/bin/nginx-prometheus-exporter \
  --nginx.scrape-uri=http://127.0.0.1:8088/nginx_status \
  --web.listen-address=:9113
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload systemd:

```bash
systemctl daemon-reload
```

Start:

```bash
systemctl enable nginx-prometheus-exporter
```

```bash
systemctl start nginx-prometheus-exporter
```

Check status:

```bash
systemctl status nginx-prometheus-exporter
```

Check logs:

```bash
journalctl -u nginx-prometheus-exporter -n 100
```

Validation:

```bash
curl http://127.0.0.1:9113/metrics
```

---

## Ten. Method 1: Prometheus Scrape Configuration

---

## Scenario 13: Prometheus static_configs

Edit configuration:

```bash
vi /etc/prometheus/prometheus.yml
```

Add job:

```yaml
scrape_configs:
  - job_name: "nginx-stub"
    static_configs:
      - targets:
          - "10.0.0.21:9113"
          - "10.0.0.22:9113"
        labels:
          env: "prod"
          service: "nginx"
          monitor_type: "stub_status"
```

Check Prometheus configuration:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Reload Prometheus:

```bash
systemctl reload prometheus
```

If reload is not supported:

```bash
systemctl restart prometheus
```

Validate from Prometheus machine:

```bash
curl http://10.0.0.21:9113/metrics
```

```bash
curl http://10.0.0.22:9113/metrics
```

Check port:

```bash
nc -zv -w 2 10.0.0.21 9113
```

---

## Eleven. Method 1: Common Metrics Explanation

Common metrics:

```text
nginx_up

nginx_connections_active

nginx_connections_accepted

nginx_connections_handled

nginx_http_requests_total

nginx_connections_reading

nginx_connections_writing

nginx_connections_waiting
```

---

## Scenario 14: Can Nginx be scraped

```promql
nginx_up
```

Meaning:

```text
1
→ exporter Successful capture stub_status

0
→ exporter Capture stub_status Failed
```

---

## Scenario 15: Active Connection Count

```promql
nginx_connections_active
```

---

## Scenario 16: QPS

Total request count:

```promql
nginx_http_requests_total
```

Requests per second:

```promql
rate(nginx_http_requests_total[1m])
```

Total QPS for all Nginx:

```promql
sum(rate(nginx_http_requests_total{job="nginx-stub"}[1m]))
```

QPS per instance:

```promql
rate(nginx_http_requests_total{job="nginx-stub"}[1m])
```

---

## Scenario 17: Reading / Writing / Waiting

```promql
nginx_connections_reading
```

```promql
nginx_connections_writing
```

```promql
nginx_connections_waiting
```

Meaning:

```text
Reading High
→ Client requests are slow to read and may be slow or unusual

Writing High
→ Responsive transmissions may be slow back end, slow client or loud Response

Waiting High
→ keepalive Free connection many times
```

---

## Twelve. Method 2: nginx-module-vts Basics

---

## Scenario 18: What is nginx-module-vts

`nginx-module-vts` is a third-party Nginx module that provides richer traffic status information.

It can provide:

```text
server Dimension Status

upstream Dimension Status

cache Dimension Status

filter Dimension Status

Request Volume

Status Code Classification

Inflow bytes

Response Time

Back-end dimension indicator
```

Common status pages:

```text
/status

/status/format/html

/status/format/json

/status/format/prometheus
```

---

## Scenario 19: Why Grafana is richer with vts scheme

Because vts metrics typically include:

```text
server zones

upstream zones

upstream backend

status code Classification

bytes in / out

response time

cache status
```

So Grafana can design more panels:

```text
Press server_name Show Request Volume

Press server_name Presentation 2xx / 3xx / 4xx / 5xx

Press upstream Show Request Volume

Press upstream Show errors

Press upstream Backend display response time

Press upstream Show traffic at backend

Press cache status Show Cache Fate
```

---

## Scenario 20: Risks of vts scheme

Main risks of vts scheme:

```text
Need to compile third-party modules

Nginx Revalidation of module compatibility during upgrade

System package installed Nginx It's not necessarily easy to load.

Production make install High risk

The long-term maintenance status of the module requires attention

Transport personnel need to know. Nginx Compile and Roll Back

Containerization Nginx Need to rebuild mirrors
```

Production recommendations:

```text
Do not try to compile directly at production nodes

Testing environment first.

Keep original Nginx Binary

Keep original configuration

Keep Rollback Scheme

Confirm. nginx -V Compile parameters

Confirm operations configuration compatibility

Confirm. reload / restart Behaviour
```

---

## Thirteen. Method 2: Compiling nginx-module-vts Installation Plan

---

## Scenario 21: Check current Nginx version and compilation parameters

Check version:

```bash
nginx -v
```

Check full compilation parameters:

```bash
nginx -V 2>&1
```

Save current compilation parameters:

```bash
nginx -V 2>&1 | tee /tmp/nginx-build-args-$(date +%F-%H%M%S).txt
```

Backup current Nginx binary:

```bash
cp -a $(which nginx) /tmp/nginx-bin-backup-$(date +%F-%H%M%S)
```

Backup configuration:

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

---

## Scenario 22: Install compilation dependencies

Ubuntu / Debian:

```bash
apt update
```

```bash
apt install -y build-essential libpcre3-dev zlib1g-dev libssl-dev wget git
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y gcc gcc-c++ make pcre-devel zlib-devel openssl-devel wget git
```

Or:

```bash
dnf install -y gcc gcc-c++ make pcre-devel zlib-devel openssl-devel wget git
```

---

## Scenario 23: Download Nginx source code and vts module

Note:

```text
Nginx Source version should be as consistent as possible with current operating version

Don't change anything.

Otherwise, it might introduce compatibility issues.
```

Example:

```bash
cd /usr/local/src
```

```bash
NGINX_VERSION=1.24.0
```

```bash
wget http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz
```

```bash
tar xf nginx-${NGINX_VERSION}.tar.gz
```

```bash
git clone https://github.com/vozlt/nginx-module-vts.git
```

---

## Scenario 24: Static compile vts module

Enter source code directory:

```bash
cd /usr/local/src/nginx-${NGINX_VERSION}
```

When configuring compilation parameters, need to retain original parameters and add:

```bash
--add-module=../nginx-module-vts
```

Example:

```bash
./configure \
  --prefix=/etc/nginx \
  --sbin-path=/usr/sbin/nginx \
  --conf-path=/etc/nginx/nginx.conf \
  --error-log-path=/var/log/nginx/error.log \
  --http-log-path=/var/log/nginx/access.log \
  --pid-path=/run/nginx.pid \
  --with-http_ssl_module \
  --with-http_stub_status_module \
  --add-module=../nginx-module-vts
```

Compile:

```bash
make
```

Do not blindly execute on production:

```bash
make install
```

Production recommendations:

```text
Let's do it on the test machine first.

Authentication nginx -t

Verify business configuration

Authentication vts Status Page

Redevelopment of production replacement options
```

---

## Scenario 25: Verify if new Nginx contains vts module

Check compilation parameters:

```bash
/usr/sbin/nginx -V 2>&1 | grep nginx-module-vts
```

Check configuration:

```bash
nginx -t
```

---

## Fourteen. Method 2: Configure vts status page

---

## Scenario 26: Basic vts configuration

Add in `http` block:

```nginx
vhost_traffic_status_zone;
vhost_traffic_status_filter_by_host on;
```

Configure status page:

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

Check configuration:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

---

## Scenario 27: Access vts status page

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

If using `nginx-vts-exporter`, usually scrape:

```text
/status/format/json
```

---

## Scenario 28: Allow monitoring machine to access vts status page

If exporter is on monitoring machine, need to allow monitoring machine access:

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

Validation:

```bash
curl http://NginxNodesIP:8089/status/format/json
```

Security recommendations:

```text
vts Do not expose the public web on the status page.

Access is only allowed to this machine or to the monitoring network

We can listen. 127.0.0.1 Or internet. IP

Do not use public network domain name exposure /status
```

---

## Fifteen. Method 2: Deploy nginx-vts-exporter

---

## Scenario 29: Run nginx-vts-exporter in binary mode

Example:

```bash
/usr/local/bin/nginx-vts-exporter \
  -nginx.scrape_uri=http://127.0.0.1:8089/status/format/json
```

Default metrics port is commonly:

```text
9913
```

Validation:

```bash
curl http://127.0.0.1:9913/metrics
```

Check metrics:

```bash
curl -s http://127.0.0.1:9913/metrics | grep nginx
```

---

## Scenario 30: Run nginx-vts-exporter in Docker

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

Check container:

```bash
docker ps | grep nginx-vts-exporter
```

Check logs:

```bash
docker logs -f nginx-vts-exporter
```

Validation:

```bash
curl http://127.0.0.1:9913/metrics
```

---

## Scenario 31: nginx-vts-exporter systemd example

Create service:

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

Reload:

```bash
systemctl daemon-reload
```

Start:

```bash
systemctl enable nginx-vts-exporter
```

```bash
systemctl start nginx-vts-exporter
```

Check status:

```bash
systemctl status nginx-vts-exporter
```

Check logs:

```bash
journalctl -u nginx-vts-exporter -n 100
```

Validation:

```bash
curl http://127.0.0.1:9913/metrics
```

---

## Sixteen. Method 2: Prometheus Scrape Configuration

---

## Scenario 32: Prometheus scrape nginx-vts-exporter

Edit Prometheus:

```bash
vi /etc/prometheus/prometheus.yml
```

Add job:

```yaml
scrape_configs:
  - job_name: "nginx-vts"
    static_configs:
      - targets:
          - "10.0.0.21:9913"
          - "10.0.0.22:9913"
        labels:
          env: "prod"
          service: "nginx"
          monitor_type: "vts"
```

Check configuration:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Reload:

```bash
systemctl reload prometheus
```

## Verification:

```bash
curl http://10.0.0.21:9913/metrics
```

```bash
curl http://10.0.0.22:9913/metrics
```

---

## 17. Method 2: Common vts Metric Directions

Different exporter versions may have different metric names, but common metric directions include:

```text
server info

server connections

server zone requests

server zone bytes

server zone cache

filter zone requests

upstream requests

upstream bytes

upstream response time

upstream backend metrics
```

Common analysis dimensions:

```text
host

code

upstream

backend

direction

status

cache status
```

Grafana panel common content:

```text
Press server_name Yes. QPS

Press server_name Yes. 2xx / 3xx / 4xx / 5xx

Press upstream Yes. QPS

Press upstream Yes. 5xx

Press upstream backend Flows

Press upstream backend Response Time

Press cache status The hit.
```

---

## 18. Method 1 and Method 2 Grafana Panel Design

---

## Scenario 33: Method 1 Grafana Panel

Suitable for displaying:

```text
Nginx up

Active connections

Reading

Writing

Waiting

QPS

Accepted connections rate

Handled connections rate

HTTP The results of the mission.

Number of days left for certificate
```

PromQL example:

```promql
nginx_up{job="nginx-stub"}
```

```promql
nginx_connections_active{job="nginx-stub"}
```

```promql
sum(rate(nginx_http_requests_total{job="nginx-stub"}[1m]))
```

```promql
nginx_connections_reading{job="nginx-stub"}
```

```promql
nginx_connections_writing{job="nginx-stub"}
```

```promql
nginx_connections_waiting{job="nginx-stub"}
```

---

## Scenario 34: Method 2 Grafana Panel

Suitable for displaying:

```text
Nginx server zone Request Volume

Nginx server zone Status Code Distribution

Nginx server zone Traffic

Nginx upstream Request Volume

Nginx upstream Status Code Distribution

Nginx upstream Backend Response Time

Nginx upstream Backend traffic

Nginx cache Hit rate

Each server_name Error Rate

Each upstream Error Rate
```

This is why:

```text
nginx-vts Programme Grafana The panels are usually more and richer.
```

---

## 19. Blackbox Exporter HTTP Probing

---

## Scenario 35: Why HTTP Probing is Still Needed

Regardless of Method 1 or Method 2, they cannot fully replace HTTP probing.

Reasons:

```text
Exporter Normal is not business URL Normal

Nginx Indicators are not normal HTTPS Certificate Normal

stub_status It's not normal. upstream Normal

vts Normal does not represent business interface return 2xx

The configuration error may affect only one server_name
```

Need to supplement:

```text
blackbox_exporter

curl Time travel.

SLB Health screening

Operations /health Interface
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

probe_ssl_earliest_cert_expiry
```

---

## 20. Certificate Expiry Monitoring

---

## Scenario 37: PromQL to Calculate Certificate Days Remaining

```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400
```

Expiry within 30 days:

```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
```

Expiry within 7 days:

```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 7
```

---

## Scenario 38: Manual Certificate Inspection

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Only checking expiry time:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -enddate
```

---

## 21. access.log Metricization Supplement

---

## Scenario 39: Why access.log is Still Needed

Even with vts, access.log remains important.

Because logs can show:

```text
Specific URI

Specific Client IP

Specific User-Agent

Specific request_id

Specific request_uri

Specific upstream_addr

Full context of a single request

Abnormal request original record
```

Recommended JSON log fields:

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

## Scenario 40: JSON access.log Example

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
    '"request_id":"$request_id"'
    '}';
```

---

## Scenario 41: Analyzing 5xx Errors

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .hostname, .remote_addr, .uri, .status, .upstream_addr, .upstream_status, .upstream_response_time] | @tsv'
```

Statistics for 5xx URLs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .uri' | sort | uniq -c | sort -nr | head
```

Statistics for 5xx upstream:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

---

## Scenario 42: Analyzing Slow Requests

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | [.time, .hostname, .uri, .status, .request_time, .upstream_response_time, .upstream_addr] | @tsv'
```

Statistics for slow request URIs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | .uri' | sort | uniq -c | sort -nr | head
```

---

## Scenario 43: Analyzing 499 Errors

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 499) | [.time, .remote_addr, .uri, .request_time] | @tsv'
```

Statistics for 499 URLs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 499) | .uri' | sort | uniq -c | sort -nr | head
```

---

## Scenario 44: Analyzing 429 Errors

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | [.time, .remote_addr, .uri, .status] | @tsv'
```

Statistics for rate-limited IPs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .remote_addr' | sort | uniq -c | sort -nr | head
```

---

## 22. error.log Monitoring

---

## Scenario 45: Key Keywords

error.log needs to focus on:

```text
connect() failed

connection refused

upstream timed out

upstream prematurely closed connection

no live upstreams

too many open files

worker_connections are not enough

client intended to send too large body

permission denied

limiting requests

limiting connections

SSL_do_handshake() failed

certificate has expired
```

---

## Scenario 46: Common error.log Commands

Check upstream errors:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

Check timeouts:

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

Check connection failures:

```bash
grep -Ei "connect\(\) failed|connection refused" /var/log/nginx/error.log | tail -n 100
```

Check no available upstream:

```bash
grep -i "no live upstreams" /var/log/nginx/error.log | tail -n 100
```

Check file descriptor exhaustion:

```bash
grep -i "too many open files" /var/log/nginx/error.log | tail -n 100
```

Check connection limit exhaustion:

```bash
grep -i "worker_connections are not enough" /var/log/nginx/error.log | tail -n 100
```

Check rate limiting:

```bash
grep -Ei "limiting requests|limiting connections" /var/log/nginx/error.log | tail -n 100
```

---

## 23. Prometheus Alerting Rules

---

## Scenario 47: Exporter Scrape Failure

```yaml
groups:
  - name: nginx.rules
    rules:
      - alert: NginxExporterDown
        expr: up{job=~"nginx-stub|nginx-vts"} == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Nginx exporter target down"
          description: "Nginx exporter target {{ $labels.instance }} is down for more than 1 minute."
```

---

## Scenario 48: stub_status Scrape Failure

```yaml
groups:
  - name: nginx.rules
    rules:
      - alert: NginxStubStatusDown
        expr: nginx_up{job="nginx-stub"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nginx stub_status unreachable"
          description: "Exporter can not scrape Nginx stub_status on {{ $labels.instance }}."
```

---

## Scenario 49: HTTP Probing Failure

```yaml
groups:
  - name: nginx.blackbox.rules
    rules:
      - alert: NginxHttpProbeFailed
        expr: probe_success{job="blackbox-nginx-http"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nginx HTTP probe failed"
          description: "HTTP probe failed for {{ $labels.target }}."
```

---

## Scenario 50: Certificate Expiry Within 30 Days

```yaml
groups:
  - name: nginx.blackbox.rules
    rules:
      - alert: NginxCertificateExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry{job="blackbox-nginx-http"} - time()) / 86400 < 30
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Nginx TLS certificate expiring soon"
          description: "TLS certificate for {{ $labels.target }} will expire in less than 30 days."
```

---

## Scenario 51: Certificate Expiry Within 7 Days

```yaml
groups:
  - name: nginx.blackbox.rules
    rules:
      - alert: NginxCertificateExpiringCritical
        expr: (probe_ssl_earliest_cert_expiry{job="blackbox-nginx-http"} - time()) / 86400 < 7
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Nginx TLS certificate expiring critical"
          description: "TLS certificate for {{ $labels.target }} will expire in less than 7 days."
```

---

## 24. Alerting Recommendations Based on Logs

The following alerts are better implemented in Loki / ELK / OpenSearch:

```text
5xx 数量或比例异常

502 数量异常

504 数量异常

499 数量异常

429 数量异常

单 IP 请求突增

某 URI 404 突增

敏感路径扫描

upstream 某节点错误率高

request_time P95 / P99 升高

upstream_response_time 升高
```

---

## Scenario 52: Loki Query for 5xx Errors

```logql
{job="nginx-access", env="prod"} | json | status >= 500
```

---

## Scenario 53: Loki Query for 502 Errors

```logql
{job="nginx-access", env="prod"} | json | status = 502
```

---

## Scenario 54: Loki Query for 504 Errors

```logql
{job="nginx-access", env="prod"} | json | status = 504
```

---

## Scenario 55: Loki Query for 429 Errors

```logql
{job="nginx-access", env="prod"} | json | status = 429
```

---

## Scenario 56: Loki Query for Slow Requests

```logql
{job="nginx-access", env="prod"} | json | request_time > 1
```

---

## 25. Nginx Observability Troubleshooting Flow

---

## Scenario 57: nginx-prometheus-exporter No Data

Troubleshoot:

```bash
docker ps | grep nginx-prometheus-exporter
```

```bash
docker logs nginx-prometheus-exporter
```

```bash
curl http://127.0.0.1:8088/nginx_status
```

```bash
curl http://127.0.0.1:9113/metrics
```

```bash
nginx -T | grep -n "stub_status" -A 10 -B 5
```

Common causes:

```text
stub_status Not configured

stub_status Port is not listening

allow / deny Intercept. exporter

Wrong access inside the container. 127.0.0.1

exporter scrape-uri Wrong

Nginx reload Not in force
```

---

## Scenario 58: nginx-vts-exporter No Data

Troubleshoot:

```bash
curl http://127.0.0.1:8089/status/format/json
```

```bash
curl http://127.0.0.1:9913/metrics
```

```bash
systemctl status nginx-vts-exporter
```

```bash
journalctl -u nginx-vts-exporter -n 100
```

```bash
nginx -T | grep -n "vhost_traffic_status" -A 10 -B 5
```

Common causes:

```text
Nginx Not compiled vts Module

vhost_traffic_status_zone Not configured

/status Path not configured

exporter scrape_uri Wrong

JSON Status Page Access Failed

vts-exporter Uncompatible version or parameters

allow / deny Intercept. exporter
```

---

## Scenario 59: HTTP Probing Failure

Troubleshoot:

```bash
curl -v https://example.com/health
```

```bash
systemctl status nginx
```

```bash
ss -lntp | grep nginx
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
nginx -T | grep -n "server_name example.com" -A 80
```

---

## Scenario 60: Sudden Increase in 5xx Errors

Troubleshoot:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .uri, .status, .upstream_addr, .upstream_status] | @tsv' | tail
```

```bash
curl -v http://BackendIP:Backend/health
```

---

## 26. Production Considerations

---

## 1. Ordinary Production Prioritize Method 1

Recommend prioritizing:

```text
stub_status + nginx-prometheus-exporter
```

Reasons:

```text
Simple

Stabilization

Low maintenance costs

No re-compilation required Nginx

Sufficient coverage of basic survival, number of connections,QPS Monitor
```

---

## 2. Consider Method 2 Only if More Grafana Panels Are Needed

If clearly needed:

```text
server_name Dimensions

upstream Dimensions

State code dimension

Backend dimension

cache Dimensions

Richer. Grafana Panel
```

Then consider:

```text
nginx-module-vts + nginx-vts-exporter
```

---

## 3. Evaluate Maintenance Costs for vts Solutions

Before deploying vts, must evaluate:

```text
Nginx Current installation

Allow rewrite

Is there a test environment?

Is there a rollback option?

Is the module compatible with the current Nginx Version

Upgrade Nginx how to handle modules

Is the team capable of compilation and containment?
```

---

## 4. Do Not Expose stub_status and vts Status Pages to Public Internet

Must restrict:

```text
Just listening. 127.0.0.1

Or allow access only to monitors.

Do not expose through a common domain name /nginx_status

Do not expose through a common domain name /status
```

---

## 5. Relying Only on Exporters Is Insufficient

Exporters cannot fully replace:

```text
HTTP Search.

HTTPS Certificate Monitor

access.log Status Code Analysis

error.log Error Key Analysis

Host CPU / Memory / Disk / Network monitoring
```

---

## 6. Logs Remain Indispensable

Metrics are suitable for trend analysis and alerts.

Logs are suitable for:

```text
Specific request

Specific URL

Specific IP

Specific upstream

Specific request_id

Specific Error Context
```

---

## 7. Control Label Cardinality in Grafana

Do not use the following fields as high cardinality labels in Prometheus / Loki:

```text
request_id

remote_addr

request_uri

args

user_agent

xff

Full URL
```

Suitable as a label:

```text
env

job

service

hostname

server_name

method

status_class

upstream
```

---

## 8. Alert thresholds must be combined with business baselines

Do not directly copy thresholds.

Example:

```text
Active connections > 10000

Reading > 500

QPS > 5000
```

These thresholds must be adjusted based on:

```text
Machine Configuration

Business peak

Historical trends

Pressure results

SLO Request
```

---

## Twenty-seven, Summary of common commands in this article

---

## stub_status Check

```bash
nginx -V 2>&1 | grep http_stub_status_module
```

```bash
curl http://127.0.0.1:8088/nginx_status
```

```bash
ss -lntp | grep ':8088'
```

```bash
nginx -T | grep -n "stub_status" -A 10 -B 5
```

---

## nginx-prometheus-exporter Check

```bash
docker ps | grep nginx-prometheus-exporter
```

```bash
docker logs -f nginx-prometheus-exporter
```

```bash
systemctl status nginx-prometheus-exporter
```

```bash
journalctl -u nginx-prometheus-exporter -n 100
```

```bash
curl http://127.0.0.1:9113/metrics
```

```bash
curl -s http://127.0.0.1:9113/metrics | grep nginx
```

```bash
ss -lntp | grep ':9113'
```

---

## vts Check

```bash
nginx -V 2>&1 | grep nginx-module-vts
```

```bash
nginx -T | grep -n "vhost_traffic_status" -A 10 -B 5
```

```bash
curl http://127.0.0.1:8089/status
```

```bash
curl http://127.0.0.1:8089/status/format/json
```

```bash
curl http://127.0.0.1:8089/status/format/prometheus
```

```bash
ss -lntp | grep ':8089'
```

---

## nginx-vts-exporter Check

```bash
docker ps | grep nginx-vts-exporter
```

```bash
docker logs -f nginx-vts-exporter
```

```bash
systemctl status nginx-vts-exporter
```

```bash
journalctl -u nginx-vts-exporter -n 100
```

```bash
curl http://127.0.0.1:9913/metrics
```

```bash
curl -s http://127.0.0.1:9913/metrics | grep nginx
```

```bash
ss -lntp | grep ':9913'
```

---

## Prometheus Check

```bash
promtool check config /etc/prometheus/prometheus.yml
```

```bash
systemctl reload prometheus
```

```bash
systemctl restart prometheus
```

```bash
curl http://10.0.0.21:9113/metrics
```

```bash
curl http://10.0.0.21:9913/metrics
```

```bash
nc -zv -w 2 10.0.0.21 9113
```

```bash
nc -zv -w 2 10.0.0.21 9913
```

---

## HTTP Probing

```bash
curl -v https://example.com/health
```

```bash
curl -I https://example.com
```

```bash
curl -I http://example.com
```

---

## Certificate Check

```bash
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

## access.log Analysis

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

## JSON access.log Analysis

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

## error.log Analysis

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

## Twenty-eight, One-sentence Summary

Nginx Prometheus monitoring typically has two common approaches:

```text
Methodology 1:

stub_status

→ nginx/nginx-prometheus-exporter

→ Prometheus

→ Grafana
```

```text
Methodology 2:

nginx-module-vts

→ /status/format/json

→ nginx-vts-exporter

→ Prometheus

→ Grafana
```

Choose between:

```text
Basic monitoring
→ Priority approach 1

Grafana The panel is richer.
→ Methodology 2 Stronger.

Lower maintenance costs
→ Methodology 1 Better.

upstream / server_name More dimensions
→ Methodology 2 Better.
```

Grafana richness:

```text
nginx-vts + nginx-vts-exporter

>

stub_status + nginx-prometheus-exporter
```

But production recommendation:

```text
Don't go blind for the panel. vts

vts Involving third-party module compilation and follow-up maintenance

General environment priority stub_status + exporter + Log Platform

Yes. upstreamI don't know.server zoneWe'll re-evaluate it when we have a breakdown of indicators. vts
```

Complete Nginx observability should include:

```text
Exporter Indicators

HTTP Search.

HTTPS Certificate Monitor

access.log Status Code Analysis

error.log Error keyword

Host CPU / Memory / Disk / Network

Grafana Watch.

Alertmanager Police!
```

Production notes:

```text
stub_status Don't expose the Internet.

vts /status Don't expose the Internet.

exporter Port only allowed Prometheus Visits

The certificate must expire with a warning.

Multiple Nginx It has to be distinguished. hostname / instance

Log label To avoid high-base segments

The warning threshold must be adjusted to the operational baseline

Indicators look at trends, logs look at details, see if the portal is really available.
```