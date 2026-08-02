# MinIO Reverse Proxy: Nginx Unified Entry, Certificates, and 9000 Port Proxy

Suggested path: 05-Storage/02-MinIO/05-MinIO Reverse Proxy: Nginx Unified Entry, Certificates, and 9000 Port Proxy.md

Tags: #MinIO #Nginx #ReverseAgent #HTTPS #S3 #ObjectStorage #UnifiedEntrance #Port9000 #Port9001 #Certificate #AdvancedSre #ProductionTransport

---

## I. Document Overview

This document is the fifth article in the MinIO module, focusing on the practical implementation of MinIO's Nginx HTTPS unified entry.

Previously completed:

- MinIO basic concepts
- Bucket / Object / Prefix
- S3 API basics
- Single-machine single-disk deployment
- Single-node multi-disk deployment
- Four-node multi-disk distributed cluster deployment
- Internal HTTP and external HTTPS entry design

This document officially enters the reverse proxy implementation phase.

This document will focus on:

    Nginx unified entry node preparation
    MinIO API HTTPS entry configuration
    MinIO Console HTTPS management entry configuration
    Certificate directory planning
    Nginx upstream configuration
    9000 API reverse proxy
    9001 Console reverse proxy
    Large file upload parameters
    S3 signature-related proxy headers
    mc accessing MinIO via HTTPS
    Bucket and object upload/download verification
    Common Nginx/MinIO entry issue troubleshooting

This document adopts the following design:

    External client access: HTTPS 443
    Nginx to MinIO backend: HTTP 9000 / 9001
    API domain: s3.minio.local
    Console domain: console.minio.local
    Backend MinIO nodes: 10.0.0.41-10.0.0.44

Notes:

    This document's experiment uses internal network domains.
    Production environments should replace with official domains and certificates.
    Production environments should not directly expose MinIO 9000 / 9001 to the public.

---

## II. Learning Objectives

After completing this document, you should be able to:

1. Understand Nginx's role in the MinIO architecture.
2. Configure HTTPS unified entry for MinIO API.
3. Configure HTTPS management entry for MinIO Console.
4. Understand the differences between 9000 API and 9001 Console proxying.
5. Master Nginx upstream proxying to multiple MinIO nodes.
6. Master Nginx parameters related to MinIO large file uploads.
7. Understand the importance of proxy headers like Host, X-Forwarded-Proto, X-Forwarded-For.
8. Connect to MinIO via HTTPS using mc.
9. Create Buckets, upload/download objects via HTTPS entry.
10. Troubleshoot common issues like 502, 413, signature errors, Console redirect errors.
11. Master MinIO entry security baseline for production environments.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This document continues with the 4-node MinIO distributed cluster:

| IP | Hostname | Role | Ports |
|---|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 | 9000 / 9001 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 | 9000 / 9001 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 | 9000 / 9001 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 | 9000 / 9001 |
| 10.0.0.45 | minio-client | mc Client | - |
| 10.0.0.46 | minio-entry | Nginx Unified Entry | 80 / 443 |

---

### 3.2 Domain Planning

Experimental domains:

| Domain | Purpose | Backend |
|---|---|---|
| s3.minio.local | S3 API entry | MinIO 9000 |
| console.minio.local | Console management entry | MinIO 9001 |

Production domain examples:

| Domain | Purpose |
|---|---|
| s3.example.com | S3 API entry |
| minio-console.example.com | Console management entry |

---

### 3.3 Image Versions

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

---

### 3.4 Operating System

Default:

    Ubuntu Server 22.04.5 LTS

Optional:

    Rocky Linux 9

This document's commands default to Ubuntu Server 22.04.5 LTS.

---

## IV. Target Architecture

### 4.1 API Access Path

    App / mc / SDK
        |
        | HTTPS 443
        v
    10.0.0.46 Nginx
        |
        | HTTP 9000
        v
    MinIO Cluster
      - 10.0.0.41:9000
      - 10.0.0.42:9000
      - 10.0.0.43:9000
      - 10.0.0.44:9000

---

### 4.2 Console Access Path

    Ops User / Browser
        |
        | HTTPS 443
        v
    10.0.0.46 Nginx
        |
        | HTTP 9001
        v
    MinIO Console
      - 10.0.0.41:9001
      - 10.0.0.42:9001
      - 10.0.0.43:9001
      - 10.0.0.44:9001

---

### 4.3 Core Principles

API and Console Separation
API faces business applications.
Console faces operations administrators.
External access uses unified HTTPS.
Internal trusted network uses HTTP.
Backend ports 9000 / 9001 should not be exposed to the public internet.
Nginx handles certificates, unified entry points, logs, and access control.

---

## Five. Pre-Operation Checks

### 5.1 Check MinIO Backend Cluster

Execute on minio-client or minio-entry:

    curl -I http://10.0.0.41:9000/minio/health/live
    curl -I http://10.0.0.42:9000/minio/health/live
    curl -I http://10.0.0.43:9000/minio/health/live
    curl -I http://10.0.0.44:9000/minio/health/live

Check readiness:

    curl -I http://10.0.0.41:9000/minio/health/ready
    curl -I http://10.0.0.42:9000/minio/health/ready
    curl -I http://10.0.0.43:9000/minio/health/ready
    curl -I http://10.0.0.44:9000/minio/health/ready

---

### 5.2 Check Console Backend

    curl -I http://10.0.0.41:9001
    curl -I http://10.0.0.42:9001
    curl -I http://10.0.0.43:9001
    curl -I http://10.0.0.44:9001

Notes:

    If Console responds, it indicates that backend port 9001 is functioning normally.
    Production environments still require restricting Console access sources.

---

### 5.3 Check Entry Node to Backend Connectivity

Execute on minio-entry:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check $ip:9000"
      nc -vz $ip 9000
    done

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check $ip:9001"
      nc -vz $ip 9001
    done

If nc is not available:

    apt update
    apt install -y netcat-openbsd

---

### 5.4 Check MinIO Cluster Status

Execute on management node:

    mkdir -p /data/minio/mc-config

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-backend http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Check status:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-backend

Requirements:

    All cluster nodes online.
    All disks online.
    No obvious offline drives.
    Ability to create Buckets and upload objects.

---

## Six. DNS Preparation

### 6.1 Linux Client hosts

Execute on minio-client or test client:

    cat >> /etc/hosts <<'EOF'
    10.0.0.46 s3.minio.local
    10.0.0.46 console.minio.local
    EOF

Verification:

    getent hosts s3.minio.local
    getent hosts console.minio.local

Expected output:

    10.0.0.46 s3.minio.local
    10.0.0.46 console.minio.local

---

### 6.2 Windows Client hosts

Edit the file:

    C:\Windows\System32\drivers\etc\hosts

Add:

    10.0.0.46 s3.minio.local
    10.0.0.46 console.minio.local

Verification:

    ping s3.minio.local
    ping console.minio.local

---

### 6.3 Production Environment DNS

Production environments should not rely on hosts files.

Use DNS resolution:

    s3.example.com -> Entry LB or Nginx VIP
    minio-console.example.com -> Entry LB or Nginx VIP

If Nginx provides high availability:

    Domain -> VIP / LB
    VIP / LB -> Multiple Nginx instances
    Nginx -> MinIO backend nodes

---

## Seven. Certificate Preparation

### 7.1 Certificate Directory

Execute on minio-entry:

    mkdir -p /etc/nginx/ssl/minio

Directory structure:

    /etc/nginx/ssl/minio/s3.minio.local.crt
    /etc/nginx/ssl/minio/s3.minio.local.key
    /etc/nginx/ssl/minio/console.minio.local.crt
    /etc/nginx/ssl/minio/console.minio.local.key

Production environments should use official certificates or internal CA certificates.

---

### 7.2 Use Official Certificates

If official certificates are available, copy them to:

    /etc/nginx/ssl/minio/s3.minio.local.crt
    /etc/nginx/ssl/minio/s3.minio.local.key
    /etc/nginx/ssl/minio/console.minio.local.crt
    /etc/nginx/ssl/minio/console.minio.local.key

Set permissions:

    chmod 600 /etc/nginx/ssl/minio/*.key
    chmod 644 /etc/nginx/ssl/minio/*.crt

Check:

    ls -l /etc/nginx/ssl/minio/

---

### 7.3 Experimental Environment Temporary Self-Signed Certificates

For experimental purposes without official certificates, temporary self-signed certificates can be generated.

High-risk warning:

Self-signed certificates are only suitable for experimentation.
Production environments should use official trusted certificates.
When using self-signed certificates, mc or browsers may prompt that the certificate is untrusted.
It is not recommended to use --insecure in production for long-term.

Generate API certificate:

    openssl req -x509 -nodes -days 365 \
      -newkey rsa:2048 \
      -keyout /etc/nginx/ssl/minio/s3.minio.local.key \
      -out /etc/nginx/ssl/minio/s3.minio.local.crt \
      -subj "/CN=s3.minio.local"

Generate Console certificate:

    openssl req -x509 -nodes -days 365 \
      -newkey rsa:2048 \
      -keyout /etc/nginx/ssl/minio/console.minio.local.key \
      -out /etc/nginx/ssl/minio/console.minio.local.crt \
      -subj "/CN=console.minio.local"

Set permissions:

    chmod 600 /etc/nginx/ssl/minio/*.key
    chmod 644 /etc/nginx/ssl/minio/*.crt

Check certificate:

    openssl x509 -in /etc/nginx/ssl/minio/s3.minio.local.crt -noout -subject -dates
    openssl x509 -in /etc/nginx/ssl/minio/console.minio.local.crt -noout -subject -dates

---

## VIII. Install Nginx

### 8.1 Ubuntu Installation

Execute on minio-entry:

    apt update
    apt install -y nginx

Start:

    systemctl enable --now nginx

Check:

    systemctl status nginx

---

### 8.2 Rocky Linux 9 Installation

If using Rocky Linux 9:

    dnf install -y nginx

Start:

    systemctl enable --now nginx

Check:

    systemctl status nginx

Firewalld allow:

    firewall-cmd --permanent --add-service=http
    firewall-cmd --permanent --add-service=https
    firewall-cmd --reload

---

### 8.3 Check Nginx Port

    ss -lntp | grep nginx

Expected:

    80 or 443 listening

---

## IX. Adjust MinIO External URL Environment Variables

### 9.1 Why It's Needed

After reverse proxying, external access address becomes:

    https://s3.minio.local
    https://console.minio.local

But backend MinIO actually listens on:

    http://10.0.0.41:9000
    http://10.0.0.41:9001

If external URL is not set, potential issues include:

    Console redirects to internal IP.
    Browser port redirection errors.
    Shared links generate internal addresses.
    SDK or mc access shows endpoint inconsistency.
    Proxyed links do not meet expectations.

Recommended settings:

    MINIO_SERVER_URL=https://s3.minio.local
    MINIO_BROWSER_REDIRECT_URL=https://console.minio.local

---

### 9.2 Need to Rebuild MinIO Container

If the previous 4 MinIO containers did not set these environment variables, it is recommended to rebuild containers during maintenance window.

High-risk warning:

    Confirm data directory mounting is correct before rebuilding containers.
    Do not delete /data/minio/disk1 and /data/minio/disk2.
    Only delete containers, not data.
    Environment variables for 4 nodes should remain consistent.
    Server endpoints for 4 nodes should remain consistent.

---

### 9.3 Stop Old Containers on Each MinIO Node

Execute on 10.0.0.41-10.0.0.44 respectively:

    docker stop minio
    docker rm minio

Confirm data directories still exist:

    ls -ld /data/minio/disk1 /data/minio/disk2

---

### 9.4 Restart MinIO with External URL

Execute the same start command on all 4 MinIO nodes:

    docker run -d \
      --name minio \
      --restart unless-stopped \
      -p 9000:9000 \
      -p 9001:9001 \
      -e MINIO_ROOT_USER=minioadmin \
      -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \
      -e MINIO_SERVER_URL=https://s3.minio.local \
      -e MINIO_BROWSER_REDIRECT_URL=https://console.minio.local \
      -v /data/minio/disk1:/data1 \
      -v /data/minio/disk2:/data2 \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
      server \
      http://10.0.0.41:9000/data1 http://10.0.0.41:9000/data2 \
      http://10.0.0.42:9000/data1 http://10.0.0.42:9000/data2 \
      http://10.0.0.43:9000/data1 http://10.0.0.43:9000/data2 \
      http://10.0.0.44:9000/data1 http://10.0.0.44:9000/data2 \
      --console-address ":9001"

Check:

    docker ps | grep minio
    docker logs --tail=100 minio

---

### 9.5 Verify Backend Recovery

Execute on minio-entry:

    curl -I http://10.0.0.41:9000/minio/health/ready
    curl -I http://10.0.0.42:9000/minio/health/ready
    curl -I http://10.0.0.43:9000/minio/health/ready
    curl -I http://10.0.0.44:9000/minio/health/ready

Check mc: /think

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  admin info minio-backend

---

## Ten. Nginx Configuration MinIO API HTTPS Entry

### 10.1 Create API Configuration File

In minio-entry execute:

    cat > /etc/nginx/conf.d/minio-s3.conf <<'EOF'
    upstream minio_s3_backend {
        least_conn;
        server 10.0.0.41:9000 max_fails=3 fail_timeout=10s;
        server 10.0.0.42:9000 max_fails=3 fail_timeout=10s;
        server 10.0.0.43:9000 max_fails=3 fail_timeout=10s;
        server 10.0.0.44:9000 max_fails=3 fail_timeout=10s;
    }

    server {
        listen 80;
        server_name s3.minio.local;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name s3.minio.local;

        ssl_certificate     /etc/nginx/ssl/minio/s3.minio.local.crt;
        ssl_certificate_key /etc/nginx/ssl/minio/s3.minio.local.key;

        client_max_body_size 0;

        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
        send_timeout 300;

        proxy_buffering off;
        proxy_request_buffering off;

        ignore_invalid_headers off;

        location / {
            proxy_pass http://minio_s3_backend;

            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Forwarded-Host $host;

            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
    EOF

---

### 10.2 Parameter Explanation

| Parameter | Function |
|---|---|
| upstream minio_s3_backend | Define MinIO API backend nodes |
| least_conn | Prioritize forwarding to backend with fewer connections |
| listen 443 ssl http2 | Enable HTTPS |
| client_max_body_size 0 | No limit on object upload size |
| proxy_buffering off | Disable response buffering, suitable for large files |
| proxy_request_buffering off | Disable request buffering, suitable for large file uploads |
| proxy_read_timeout 300 | Increase long connection read timeout |
| proxy_set_header Host $http_host | Preserve Host, avoid S3 signature issues |
| X-Forwarded-Proto https | Inform backend that external protocol is HTTPS |

---

### 10.3 Configuration Check

    nginx -t

If configuration is correct:

    syntax is ok
    test is successful

Reload:

    systemctl reload nginx

Check port:

    ss -lntp | grep nginx

---

## Eleven. Nginx Configuration MinIO Console HTTPS Entry

### 11.1 Create Console Configuration File

In minio-entry execute:

    cat > /etc/nginx/conf.d/minio-console.conf <<'EOF'
    upstream minio_console_backend {
        least_conn;
        server 10.0.0.41:9001 max_fails=3 fail_timeout=10s;
        server 10.0.0.42:9001 max_fails=3 fail_timeout=10s;
        server 10.0.0.43:9001 max_fails=3 fail_timeout=10s;
        server 10.0.0.44:9001 max_fails=3 fail_timeout=10s;
    }

    server {
        listen 80;
        server_name console.minio.local;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name console.minio.local;

ssl_certificate     /etc/nginx/ssl/minio/console.minio.local.crt;
        ssl_certificate_key /etc/nginx/ssl/minio/console.minio.local.key;

        client_max_body_size 0;

        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
        send_timeout 300;

        location / {
            proxy_pass http://minio_console_backend;

            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Forwarded-Host $host;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
    EOF

---

### 11.2 Recommended Access Restrictions for Console

It is recommended to add source IP restrictions for the Console in production environments.

Example:

    allow 10.0.0.0/24;
    deny all;

This can be placed in the Console's server or location block.

Example snippet:

    location / {
        allow 10.0.0.0/24;
        deny all;

        proxy_pass http://minio_console_backend;
        ...
    }

Notes:

    Experimental environments can initially have no restrictions.
    The Console should not be open to the public in production environments.
    It is more recommended to access via VPN, bastion host, or operations network segment.

---

### 11.3 Check and Reload

    nginx -t
    systemctl reload nginx

Check logs:

    tail -f /var/log/nginx/access.log
    tail -f /var/log/nginx/error.log

---

## Twelve、HTTPS Entry Verification

### 12.1 Verify API HTTPS

Execute on the client side:

    curl -k -I https://s3.minio.local/minio/health/live
    curl -k -I https://s3.minio.local/minio/health/ready

If using a formal trusted certificate, do not use -k:

    curl -I https://s3.minio.local/minio/health/live

Notes:

    -k is only suitable for self-signed certificate experiments.
    It is not recommended to ignore certificate validation in production.

---

### 12.2 Verify Console HTTPS

Access via browser:

    https://console.minio.local

If using a self-signed certificate, the browser will prompt that it is untrusted.

Login:

    Username: minioadmin
    Password: MinioAdmin@123456

Production reminder:

    The root user is only for management.
    Do not hand over the root user to business applications.
    Business applications should use independent users and Policies.

---

### 12.3 View Nginx Logs

After access, check in minio-entry:

    tail -f /var/log/nginx/access.log

Error logs:

    tail -f /var/log/nginx/error.log

If 502 occurs, focus on checking:

    Whether the backend 9000 / 9001 is reachable.
    Whether the upstream IP is written incorrectly.
    Whether the MinIO container is running.
    Whether the firewall is blocking.

---

## Thirteen、mc Access via HTTPS Entry

### 13.1 Configure alias when using a formal certificate

If the certificate is trusted:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-https

---

### 13.2 Configure alias when using a self-signed certificate

If using a self-signed certificate for experimentation, mc may report certificate untrustworthiness.

You can temporarily use --insecure:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-https

Production reminder:

    --insecure is only suitable for experiments.
    Production should use trusted certificates.
    Production should not long-term skip certificate validation.

---

### 13.3 Create Bucket Verification

Create Bucket: /think

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mb minio-https/proxy-demo

If using a formal certificate, remove --insecure:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    mb minio-https/proxy-demo

Check:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure ls minio-https

---

### 13.4 Object Upload Verification

Create test file:

  mkdir -p /tmp/minio-proxy-demo

  echo "hello minio nginx https proxy" > /tmp/minio-proxy-demo/hello.txt

Upload:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-proxy-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure cp /demo/hello.txt minio-https/proxy-demo/hello.txt

Check:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure ls minio-https/proxy-demo

Download:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-proxy-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure cp minio-https/proxy-demo/hello.txt /demo/hello-download.txt

Verify:

  cat /tmp/minio-proxy-demo/hello-download.txt

---

### 13.5 Large File Upload Verification

Create 200M file:

  dd if=/dev/zero of=/tmp/minio-proxy-demo/file-200m.bin bs=1M count=200

Upload:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-proxy-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure cp /demo/file-200m.bin minio-https/proxy-demo/file-200m.bin

Check object:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure stat minio-https/proxy-demo/file-200m.bin

If upload fails, focus check:

  client_max_body_size
  proxy_request_buffering
  proxy_read_timeout
  proxy_send_timeout
  Nginx error.log
  MinIO backend health status

---

## Fourteen, AWS CLI Verification

### 14.1 Configure AWS CLI

If the client has installed AWS CLI, configure:

  aws configure --profile minio-lab

Input:

  AWS Access Key ID: minioadmin
  AWS Secret Access Key: MinioAdmin@123456
  Default region name: us-east-1
  Default output format: json

Set Path-Style:

  aws configure set profile.minio-lab.s3.addressing_style path

---

### 14.2 Access Bucket

If using self-signed certificate, may need to add --no-verify-ssl.

Check Bucket:

  aws --profile minio-lab \
    --endpoint-url https://s3.minio.local \
    --no-verify-ssl \
    s3 ls

Upload:

  aws --profile minio-lab \
    --endpoint-url https://s3.minio.local \
    --no-verify-ssl \
    s3 cp /tmp/minio-proxy-demo/hello.txt s3://proxy-demo/aws-hello.txt

Download:

aws --profile minio-lab \
  --endpoint-url https://s3.minio.local \
  --no-verify-ssl \
  s3 cp s3://proxy-demo/aws-hello.txt /tmp/minio-proxy-demo/aws-hello-download.txt

View:

  cat /tmp/minio-proxy-demo/aws-hello-download.txt

Production Reminder:

  --no-verify-ssl is only suitable for experimental self-signed certificates.
  Production must use trusted certificates and should not skip SSL verification.

---

## FifteenI don't know.Nginx Access Control Reinforcement

### 15.1 Restrict Console Sources

In production, it is recommended to restrict sources for console.minio.local.

Modify the location configuration in the Console:

  location / {
      allow 10.0.0.0/24;
      deny all;

      proxy_pass http://minio_console_backend;

      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto https;
      proxy_set_header X-Forwarded-Host $host;

      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
  }

Check:

  nginx -t
  systemctl reload nginx

Explanation:

  10.0.0.0/24 is an experimental network segment.
  Production should replace with operations network segment, VPN network segment, or bastion host export IP.

---

### 15.2 Not Recommended to Restrict API Sources Too Narrowly

The API faces business applications, and whether to restrict sources depends on the architecture.

Recommendations:

  Must use HTTPS when open to the public.
  Can combine with WAF / LB / security group.
  For internal systems, can restrict business network segments.
  Do not open meaningless management ports.
  Do not allow 9000 backend to bypass the entrance and expose directly.

---

### 15.3 Hide Nginx Version

Can set in the http segment of /etc/nginx/nginx.conf:

  server_tokens off;

Check:

  nginx -t
  systemctl reload nginx

---

## SixteenI don't know.Firewall Recommendations

### 16.1 minio-entry Firewall

Open the following ports on the entry node:

  80
  443
  22, only from operations sources

Ubuntu ufw example:

  ufw allow 80/tcp
  ufw allow 443/tcp
  ufw allow from 10.0.0.0/24 to any port 22 proto tcp
  ufw status numbered

---

### 16.2 MinIO Backend Node Firewall

Backend nodes are recommended to:

  9000 only allow minio-entry and necessary internal network access
  9001 only allow operations network segment access
  22 only allow operations network segment access

Example:

  ufw allow from 10.0.0.46 to any port 9000 proto tcp
  ufw allow from 10.0.0.0/24 to any port 9001 proto tcp
  ufw allow from 10.0.0.0/24 to any port 22 proto tcp
  ufw status numbered

Experimental environments can initially loosen, production must tighten.

---

## SeventeenI don't know.Common Issue Troubleshooting

### 17.1 Nginx 502 Bad Gateway

Symptoms:

  Accessing https://s3.minio.local returns 502

Troubleshoot:

  systemctl status nginx
  nginx -t
  tail -f /var/log/nginx/error.log

Check backend:

  curl -I http://10.0.0.41:9000/minio/health/live
  curl -I http://10.0.0.42:9000/minio/health/live
  curl -I http://10.0.0.43:9000/minio/health/live
  curl -I http://10.0.0.44:9000/minio/health/live

Common causes:

  Backend MinIO container is not running.
  Upstream IP is incorrect.
  Port 9000 is unreachable.
  Firewall interception.
  Nginx cannot resolve backend domain.
  All backend nodes are abnormal.

---

### 17.2 Upload Large File Returns 413

Symptoms:

  Request Entity Too Large

Cause:

  client_max_body_size is too small.

Solution:

  client_max_body_size 0;

Or set explicit size:

  client_max_body_size 10G;

Reload:

  nginx -t
  systemctl reload nginx

---

### 17.3 Upload Large File Interrupted

Possible causes:

  proxy_read_timeout is too short.
  proxy_send_timeout is too short.
  proxy_request_buffering is not disabled.
  Network interruption.
  Backend MinIO node is abnormal.
  Disk space is insufficient.

Check:

  tail -f /var/log/nginx/error.log
  docker logs minio
  df -hT
  docker run --rm -v /data/minio/mc-config:/root/.mc registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z --insecure admin info minio-https

---

### 17.4 mc Reports Certificate Error

If it's a self-signed certificate, mc may report:

  certificate signed by unknown authority

Experimental handling:

  Use --insecure

Production handling:

Use formal trusted certificates.
Or import the enterprise CA into the client's trust chain.
Not recommended for long-term use in production --insecure.

---

### 17.5 S3 SignatureDoesNotMatch

Phenomenon:

    SignatureDoesNotMatch
    The request signature we calculated does not match the signature you provided

Common causes:

    Host header is rewritten by Nginx.
    Client endpoint does not match the actual access domain.
    HTTP/HTTPS protocol inconsistency.
    Application uses virtual-host-style but domain and certificate do not support.
    Proxy is not correctly set X-Forwarded-Proto.
    Incorrect AccessKey / SecretKey used.

Handling directions:

    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto https;
    endpoint uses https://s3.minio.local;
    Use path-style in experimental phase.
    Check AccessKey and SecretKey.
    Check system time synchronization.

---

### 17.6 Console Redirects to Internal IP

Phenomenon:

    After accessing https://console.minio.local, redirects to http://10.0.0.41:9001

Possible causes:

    MINIO_BROWSER_REDIRECT_URL not set.
    MINIO_SERVER_URL not set.
    Nginx forwarding headers are incomplete.
    Old container environment variables not updated.

Handling:

    Check MinIO container environment variables.
    Reconfigure using the following commands:
      MINIO_SERVER_URL=https://s3.minio.local
      MINIO_BROWSER_REDIRECT_URL=https://console.minio.local

Check container environment variables:

    docker inspect minio | grep -E 'MINIO_SERVER_URL|MINIO_BROWSER_REDIRECT_URL'

---

### 17.7 Console Page Blank or WebSocket Exception

Possible causes:

    Nginx does not set Upgrade header.
    Connection header is incorrect.
    Proxy timeout.
    Console upstream is unreachable.

Check if the following are included in Console configuration:

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

Check:

    tail -f /var/log/nginx/error.log

---

## EighteenI don't know.Production Environment Notes

### 18.1 Certificates

Production requirements:

    Use formal trusted certificates.
    Certificate domain matches access domain.
    Certificate private key permissions tightened.
    Certificate expiration included in monitoring.
    Do not use --insecure.
    Do not expose long-term self-signed certificates to business.

---

### 18.2 Entry High Availability

This document uses a single Nginx:

    10.0.0.46 minio-entry

Production should consider:

    Dual Nginx
    Keepalived VIP
    Cloud load balancer
    Hardware load balancer
    DNS disaster recovery
    Entry layer health check

Otherwise:

    MinIO backend is multi-node.
    But the entry Nginx remains a single point.

---

### 18.3 Console Security

Production recommendations:

    Console should not be directly exposed to public internet.
    Only allow operations network segments.
    Use HTTPS.
    Use strong passwords.
    Do not share root user.
    Log management operations.
    Revoke accounts promptly for departing personnel.
    Subsequent use independent users with minimal permissions policy.

---

### 18.4 Backend Port Security

Production requirements:

    9000 should not be directly exposed to public internet.
    9001 should not be directly exposed to public internet.
    9000 should only allow access from entry layer and necessary business network segments.
    9001 should only allow access from operations network segments.
    Firewall and security group rules have records.
    Changes to entry rules should have rollback plans.

---

### 18.5 Nginx Logs

Production recommendations:

    Enable access.log.
    Retain error.log.
    Integrate logs with Loki / ELK.
    Monitor 4xx / 5xx.
    Monitor failed large file uploads.
    Monitor source IP.
    Monitor abnormal console login requests.

---

## NineteenI don't know.Verification Checklist

### 19.1 Backend Verification

| Check item | Command | Result |
|---|---|---|
| MinIO live | curl /minio/health/live |  |
| MinIO ready | curl /minio/health/ready |  |
| 9000 port | nc -vz IP 9000 |  |
| 9001 port | nc -vz IP 9001 |  |
| mc admin info | mc admin info |  |

---

### 19.2 Nginx Verification

| Check item | Command | Result |
|---|---|---|
| Configuration syntax | nginx -t |  |
| Service status | systemctl status nginx |  |
| 443 listening | ss -lntp grep nginx |  |
| API HTTPS | curl https://s3.minio.local |  |
| Console HTTPS | Browser access console.minio.local |  |
| Nginx logs | access.log / error.log |  |

---

### 19.3 S3 Function Verification

| Check item | Command | Result |
|---|---|---|
| alias set | mc alias set |  |
| Cluster info | mc admin info |  |
| Create Bucket | mc mb |  |
| Upload small file | mc cp |  |
| Download small file | mc cp |  |
| Upload large file | mc cp 200M |  |
| View object | mc stat |  |

---

## TwentyI don't know.Experiment Cleanup

### 20.1 Delete Test Bucket

High-risk warning:

    Deleting a Bucket will delete objects.
    Only allow cleanup of experimental Buckets.
    Production environment must confirm backups and business ownership.

Delete objects: /think

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure rm --recursive --force minio-https/proxy-demo

Delete Bucket:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure rb minio-https/proxy-demo

---

### 20.2 Clean Up Local Test Files

    rm -rf /tmp/minio-proxy-demo

---

### 20.3 Should You Clean Up Nginx Configuration

If it's just an experiment and no longer needed:

    rm -f /etc/nginx/conf.d/minio-s3.conf
    rm -f /etc/nginx/conf.d/minio-console.conf

Reload:

    nginx -t
    systemctl reload nginx

Do not arbitrarily delete entry configurations in production environments.

---

## Twenty-oneI don't know.Interview Answering Strategy

If asked in an interview:

    Why is Nginx placed in front of MinIO? How to configure reverse proxy?

You can answer:

    MinIO backend nodes typically listen on 9000 API port and 9001 Console port. In production environments, it's not recommended to let business directly access a backend node's HTTP 9000, nor to expose Console 9001 to the public internet.
    I would place Nginx or LB as a unified entry point. Provide HTTPS externally, for example, s3.example.com proxies to multiple MinIO nodes' 9000 port, Console uses an independent domain, for example, minio-console.example.com, proxies to 9001, and restrict access to the operation network segment.
    Several points need attention in Nginx configuration: First, client_max_body_size should be opened or set according to business needs, otherwise large object uploads will report 413; Second, proxy_request_buffering and proxy_buffering are recommended to be disabled to avoid large file uploads/downloads being affected by Nginx buffering; Third, retain the Host header and set X-Forwarded-Proto, X-Forwarded-For headers, otherwise S3 signature errors or Console redirection anomalies may occur.
    It's also recommended to set MINIO_SERVER_URL to the external API address and MINIO_BROWSER_REDIRECT_URL to the Console address on the MinIO container side to avoid redirection to internal IP or errors after reverse proxy.
    If internal Nginx to MinIO nodes are in a trusted internal network, HTTP can be used to reduce certificate management complexity; but external client access must use HTTPS to protect AccessKey signing and object data. Production environments also need to consider Nginx high availability, certificate expiration monitoring, access logs, security groups, and Console access control.

---

## Twenty-twoI don't know.Summary of This Chapter

This article completed the hands-on practice of setting up a unified HTTPS entry point for MinIO with Nginx:

1. MinIO API uses 9000 port.
2. MinIO Console uses 9001 port.
3. Production environments should not expose HTTP 9000 directly to the public internet.
4. Production environments should not expose Console 9001 directly to the public internet.
5. Nginx can serve as a unified HTTPS entry point.
6. API domain is recommended to be s3.example.com or s3.minio.local.
7. Console domain is recommended to be minio-console.example.com or console.minio.local.
8. Nginx API upstream should proxy to 10.0.0.41-10.0.0.44's 9000.
9. Nginx Console upstream should proxy to 10.0.0.41-10.0.0.44's 9001.
10. Large file uploads need to pay attention to parameters like client_max_body_size, proxy_request_buffering, proxy_read_timeout, etc.
11. S3 signature scenarios need to retain the Host header.
12. It's recommended to set MINIO_SERVER_URL and MINIO_BROWSER_REDIRECT_URL after reverse proxy.
13. mc can access MinIO through HTTPS entry point.
14. Self-signed certificate experiments can temporarily use --insecure, but should not be used long-term in production.
15. Console entry should restrict operation network segments.
16. Backend 9000/9001 should be restricted by firewall or security group.
17. The entry layer itself also needs high availability design.
18. Future will continue to learn mc client tools, bucket management, and object operations.

---

## Twenty-threeI don't know.Reference Documents

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Docker Deployment Documentation:

    https://min.io/docs/minio/container/index.html

MinIO Reverse Proxy Related Documentation:

    https://min.io/docs/minio/linux/integrations/setup-nginx-proxy-with-minio.html

MinIO Console Documentation:

    https://min.io/docs/minio/linux/administration/minio-console.html

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO Management Command Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc-admin.html

MinIO Identity and Access Management:

    https://min.io/docs/minio/linux/administration/identity-access-management.html

Nginx Official Documentation:

    https://nginx.org/en/docs/

AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html