# MinIO Reverse Proxy: Nginx Unified Entrance, Certificate, and 9000 Port Proxy

Recommended Path: 05-Storage/02-MinIO/05-MinIO Reverse Proxy: Nginx Unified Entrance, Certificate, and 9000 Port Proxy.md

Tags: #MinIO #Nginx #Reverse Proxy #HTTPS #S3 #Object Storage #Unified Entrance #9000 Port #9001 Port #Certificate #Advanced SRE #Production Operations

---

## I. Document Description

This article is the fifth in the MinIO series, focusing on the practical implementation of a unified Nginx HTTPS entrance for MinIO.

Previous steps have included:

- Basic concepts of MinIO
- Bucket / Object / Prefix
- Basics of S3 API
- Single-machine single-disc deployment
- Single-node multi-disk deployment
- 4-node multi-disk distributed cluster deployment
- Design of internal HTTP and external HTTPS entrances

This article now enters the practical phase of setting up a reverse proxy.

Key tasks include:

    Preparation of the Nginx unified entrance node
    Configuration of the MinIO API HTTPS entrance
    Configuration of the MinIO Console HTTPS entrance
    Planning of the certificate directory
    Nginx upstream configuration
    9000 API reverse proxy
    9001 Console reverse proxy
    Large file upload parameters
    S3 signature-related proxy headers
    Accessing MinIO through HTTPS using mc
    Verification of bucket and object uploads and downloads
    Troubleshooting common Nginx/MinIO entrance issues

The design adopted in this article is:

    External client access: HTTPS 443
    Nginx to MinIO backend: HTTP 9000 / 9001
    API domain name: s3.minio.local
    Console domain name: console.minio.local
    Backend MinIO nodes: 10.0.0.41-10.0.0.44

Notes:

    This experiment uses an internal domain name.
    In a production environment, replace with official domain names and certificates.
    It is not recommended to directly expose MinIO's 9000/9001 ports to the public network in a production setting.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the role of Nginx in the MinIO architecture.
2. Configure a unified HTTPS entrance for the MinIO API.
3. Set up an HTTPS management entrance for the MinIO Console.
4. Distinguish between the 9000 API and 9001 Console proxy functions.
5. Master how to use Nginx upstream proxies to connect to multiple MinIO nodes.
6. Understand the relevant Nginx parameters for large file uploads with MinIO.
7. Recognize the importance of proxy headers such as Host, X-Forwarded-Proto, and X-Forwarded-For.
8. Use mc to connect to MinIO via the HTTPS entrance.
9. Create buckets, upload objects, and download objects through the HTTPS entrance.
10. Troubleshoot common issues such as 502 errors, 413 errors, signature errors, and Console redirection errors.
11. Understand the security baseline for MinIO entrances in a production environment.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This article continues with the 4-node MinIO distributed cluster:

| IP | Hostname | Role | Ports |
|---|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 | 9000 / 9001 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 | 9000 / 9001 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 | 9000 / 9001 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 | 9000 / 9001 |
| 10.0.0.45 | minio-client | mc Client | - |
| 10.0.0.46 | minio-entry | Nginx Unified Entrance | 80 / 443 |

---

### 3.2 Domain Name Planning

Experimental domain names:

| Domain Name | Purpose | Backend |
|---|---|---|
| s3.minio.local | S3 API Entrance | MinIO 9000 |
| console.minio.local | Console Management Entrance | MinIO 9001 |

Example production domain names:

| Domain Name | Purpose |
|```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  admin info minio-backend
```registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
server \
http://10.0.0.41:9000/data1 http://10.0.0.41:9000/data2 \
http://10.0.0.42:9000/data1 http://10.0.0.42:9000/data2 \
http://10.0.0.43:9000/data1 http://10.0.0.43:9000/data2 \
http://10.0.0.44:9000/data1 http://10.0.0.44:9000/data2 \
--console-address ":9001"

View:

    docker ps | grep minio
    docker logs --tail=100 minio

---

### 9.5 Verify Backend Recovery

Execute the following commands in minio-entry:

    curl -I http://10.0.0.41:9000/minio/health/ready
    curl -I http://10.0.0.42:9000/minio/health/ready
    curl -I http://10.0.0.43:9000/minio/health/ready
    curl -I http://10.0.0.44:9000/minio/health/ready

View mc:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-backend

---

## Section 10: Configuring MinIO API HTTPS Access with Nginx

### 10.1 Creating the API Configuration File

Execute the following command in minio-entry:

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
| upstream minio_s3_backend | Defines the MinIO API backend servers |
| least_conn | Prioritizes connections to backends with fewer active connections |
| listen 443 ssl http2 | Enables HTTPS support |
| client_max_body_size 0 | Allows unlimited object sizes for uploads |
| proxy_buffering off | Disables response buffering, suitable for large files |
| proxy_request_buffering off | Disables request buffering, suitable for large file uploads |
| proxy_read_timeout 300 | Increases the timeout for reading in long connections |
| proxy_set_header Host $http_host | Preserves the original Host header to avoid S3 signature issues |
| X-Forwarded-Proto httpsproxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
}
}
EOF

---

### 11.2 Recommendations for Console Access Restrictions

In a production environment, it is recommended to restrict console access based on the source IP address.

Example:

    allow 10.0.0.0/24;
    deny all;

These rules can be applied in the server or location block of the Nginx configuration file.

Example snippet:

    location / {
        allow 10.0.0.0/24;
        deny all;

        proxy_pass http://minio_console_backend;
        ...
    }

Explanation:

    In a testing environment, access restrictions may not be necessary.
    In a production environment, it is not recommended to expose the console directly over the public internet.
    It is more secure to use VPNs, bastion hosts, or dedicated network segments for console access.

---

### 11.3 Checking and Reloading

    Run the following commands to check Nginx configuration and reload it:

    nginx -t
    systemctl reload nginx

To monitor logs, use the following commands:

    tail -f /var/log/nginx/access.log
    tail -f /var/log/nginx/error.log

---

## Chapter 12: HTTPS Access Verification

### 12.1 Verifying API HTTPS

On the client side, execute the following commands:

    curl -k -I https://s3.minio.local/minio/health/live
    curl -k -I https://s3.minio.local/minio/health/ready

If using a legitimate and trusted certificate, you should remove the `-k` option:

    curl -I https://s3.minio.local/minio/health/live

Explanation:

    The `-k` option is suitable only for testing with self-signed certificates.
    In a production environment, it is essential to verify the certificate to ensure security.

---

### 12.2 Verifying Console HTTPS

Access the console using a browser:

    https://console.minio.local

If a self-signed certificate is used, the browser may warn that the certificate is not trusted.

To log in, use the following credentials:

    Username: minioadmin
    Password: MinioAdmin@123456

Production note:

    The `root` user should be used only for administrative purposes.
    Do not grant `root` access to business users.
    Business applications should have their own dedicated users and security policies.

---

### 12.3 Checking Nginx Logs

To view logs, navigate to the `/minio-entry` directory and use the following commands:

    tail -f /var/log/nginx/access.log
    tail -f /var/log/nginx/error.log

If a 502 error occurs, check the following aspects:

    Ensure that the backend services (ports 9000/9001) are running.
    Verify whether the upstream IP address is correctly specified in the Nginx configuration.
    Confirm that the MinIO container is functioning properly.
    Check if there are any firewall restrictions blocking the connections.

---

## Chapter 13: Accessing mc via HTTPS

### 13.1 Configuring an Alias Using a Legitimate Certificate

If you have a trusted certificate, use the following command to configure an alias in Docker:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

To check the configuration, run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-https

---

### 13.2 Configuring an Alias Using a Self-Signed Certificate

If you are using a self-signed certificate, the `mc` command may report that the certificate is not trusted.

You can temporarily use the `--insecure` option:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

To check the configuration, run:

    docker-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure stat minio-https/proxy-demo/file-200m.bin

If the upload fails, focus on checking the following:

    client_max_body_size
    proxy_request_buffering
    proxy_read_timeout
    proxy_send_timeout
    Nginx error.log
    Health status of the MinIO backend

---

## Section 14: Verifying Using aws cli

### 14.1 Configuring aws cli

If the client has aws cli installed, you can configure it as follows:

    aws configure --profile minio-lab

Enter the following information:

    AWS Access Key ID: minioadmin
    AWS Secret Access Key: MinioAdmin@123456
    Default region name: us-east-1
    Default output format: json

Set the Path-Style:

    aws configure set profile.minio-lab.s3.addressing_style path

---

### 14.2 Accessing the Bucket

If using a self-signed certificate, you may need to add --no-verify-ssl.

To view the contents of the bucket:

    aws --profile minio-lab \
      --endpoint-url https://s3.minio.local \
      --no-verify-ssl \
      s3 ls

To upload a file:

    aws --profile minio-lab \
      --endpoint-url https://s3.minio.local \
      --no-verify-ssl \
      s3 cp /tmp/minio-proxy-demo/hello.txt s3://proxy-demo/aws-hello.txt

To download a file:

    aws --profile minio-lab \
      --endpoint-url https://s3.minio.local \
      --no-verify-ssl \
      s3 cp s3://proxy-demo/aws-hello.txt /tmp/minio-proxy-demo/aws-hello-download.txt

To view the downloaded file:

    cat /tmp/minio-proxy-demo/aws-hello-download.txt

Production note:

    --no-verify-ssl is only suitable for testing with self-signed certificates. In production, you must use trusted certificates and not skip SSL verification.

---

## Section 15: Strengthening Nginx Access Control

### 15.1 Restricting Console Access Sources

In production, it is recommended to restrict access to console.minio.local.

Modify the location block in the console configuration:

    location / {
        allow 10.0.0.0/24;
        deny all;

        proxy_pass http://minio-console_backend;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

Check the configuration:

    nginx -t
    systemctl reload nginx

Note:

    10.0.0.0/24 is a test network range. In production, you should replace it with your operational network range, VPN network range, or bastion host IP address.

---

### 15.2 Not Recommended: Narrowly Restricting API Access Sources

APIs are designed for business applications, so whether to restrict access sources depends on the specific business architecture.

Recommendations:

    When opening APIs to the public internet, HTTPS must be used.
    You can use WAF, LB, or security groups for additional protection.
    For internal systems, you can limit access to specific business network ranges.
    Do not expose unnecessary management ports.
    Ensure that the 9000 backend port is not directly accessible from external sources.

---

### 15.3 Hiding the Nginx Version

You can set this in the http section of /etc/nginx/nginx.conf:

    server_tokens off;

Check the configuration:

    nginx -t
    systemctl reload nginx

---

## Section 16: Firewall Recommendations

### 16.1 minio-entry Firewall

For the entry node, open the following ports:

    80
    443
    22 (only for operational access)

Example for Ubuntu ufw:

    ufw allow 80/tcp
    ufw allow 443/tcp
    ufw allow from 10.0.0.0/24 to any port 22 proto tcp
    ufw status numbered

---

### 16.2 MinIO Backend Node Firewall

For the backend nodes, it is recommended to open:

    Port The client endpoint does not match the actual domain name being accessed.
The HTTP/HTTPS protocols are inconsistent.
The application uses a virtual-host-style configuration, but the domain name and certificate do not support it.
The proxy is not correctly setting the X-Forwarded-Proto header.
The wrong AccessKey/SecretKey is being used.

Action Steps:

Set the `Host` header in the proxy: `proxy_set_header Host $http_host;`
Set the `X-Forwarded-Proto` header to `https`: `proxy_set_header X-Forwarded-Proto https;`
Use `https://s3.minio.local` as the endpoint.
Use a path-style configuration during the experimental phase.
Check the AccessKey and SecretKey values.
Ensure that the system time is synchronized.

---

### 17.6 Console Redirects to Internal IP

Issue:

Accessing `https://console.minio.local` results in a redirect to `http://10.0.0.41:9001`.

Possible Causes:

The `MINIO_BROWSER_REDIRECT_URL` variable is not set.
The `MINIO_SERVER_URL` variable is not set.
The Nginx forwarding headers are incomplete.
Old container environment variables have not been updated.

Solution:

Check the MinIO container's environment variables.
Set them according to the instructions in this document:
```
MINIO_SERVER_URL=https://s3.minio.local
MINIO_BROWSER_REDIRECT_URL=https://console.minio.local
```

To check the container environment variables, use the following command:
```
docker inspect minio | grep -E 'MINIO_SERVER_URL|MINIO-browser_redirect_URL'
```

---

### 17.7 Console Pages Are Blank or WebSocket Errors Occur

Possible Causes:

Nginx is not setting the `Upgrade` header.
The `Connection` header is incorrect.
The proxy is timing out.
There is an issue with the upstream connection to the Console.

Check whether the following lines are included in the Nginx configuration:
```
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

Also, monitor the `/var/log/nginx/error.log` file for any related errors.

---

## Chapter Eighteen: Considerations for Production Environment

### 18.1 Certificates

Production Requirements:

Use official and trusted certificates.
Ensure that the certificate domain name matches the access domain name.
Improve the security settings of the certificate private key.
Monitor certificate expiration dates.
Do not use the `--insecure` option.
Avoid using long-term self-signed certificates in production environments.

---

### 18.2 High Availability at the Entrance Layer

This document uses a single Nginx server:
```
10.0.0.46 minio-entry
```

For production, consider the following options:

Use multiple Nginx servers.
Set up a Keepalived VIP for load balancing.
Utilize cloud or hardware load balancing solutions.
Implement DNS disaster recovery measures.
Perform health checks at the entrance layer.

Alternatively, if there are multiple MinIO backend nodes, ensure that the Nginx server serving as the entrance is also highly available.

---

### 18.3 Console Security

Production Recommendations:

Do not expose the Console directly to the public internet.
Only allow access from the operations network segment.
Use HTTPS for secure communications.
Set strong passwords.
Avoid sharing the root user account.
Keep logs of all management operations.
Revoke accounts of departing employees promptly.
Implement independent user accounts and minimum permission policies.

---

### 18.4 Security for Backend Ports

Production Requirements:

Do not expose ports 9000 and 9001 directly to the public internet.
Allow access to port 9000 only from the entrance layer and necessary business network segments.
Allow access to port 9001 only from the operations network segment.
Ensure that firewall and security group rules are properly configured.
Have a backup plan in place in case of any changes to the entry rules.

---

### 18.5 Nginx Logs

Production Recommendations:

Enable the `access.log` file for logging purposes.
Retain the `error.log` file for troubleshooting.
Integrate log data into systems like Loki or ELK for analysis.
Pay special attention to errors with HTTP status codes 4xx and 5xx, as well as failures in large-file uploads.
Monitor source IP addresses and any suspicious login attempts from the Console.

---

## Chapter Nineteen: Verification Checklist

### 19.1 Backend Verification

| Check Item | Command | Result |
|-------------|---------|----------|
| MinIO health check | curl /minio/health/live | ... |
| MinIO readiness check | curl /minio/health/ready | ... |
| Port 9000 availability | nc -vz IP 9000 | ... |
| Port 9001 availability | nc -vz IP 9001 |4. It is not recommended to directly expose Console 9001 to the public internet in production environments.
5. Nginx can be used as a unified HTTPS entry point.
6. It is suggested that the API domain name be set to s3.example.com or s3.minio.local.
7. The Console domain name should be set to minio-console.example.com or console.minio.local.
8. The Nginx API upstream should proxy requests to port 9000 on servers 10.0.0.41-10.0.0.44.
9. The Nginx Console upstream should also proxy requests to port 9001 on the same servers.
10. When handling large file uploads, attention should be paid to parameters such as client_max_body_size, proxy_request_buffering, and proxy_read_timeout.
11. In S3 signature scenarios, it is necessary to retain the Host header.
12. After setting up a reverse proxy, it is recommended to configure MINIO_SERVER_URL and MINIO_BROWSER_REDIRECT_URL.
13. The mc client can access MinIO through an HTTPS interface.
14. For experimental purposes with self-signed certificates, --insecure can be temporarily used, but this should not be adopted in production settings.
15. The Console access should be restricted to specific operational network segments only.
16. The backend ports 9000 and 9001 should have their sources filtered through firewalls or security groups.
17. The entry layer itself must also be designed for high availability.
18. Further study will focus on the mc client tool, Bucket management, and object operations.

---

## Chapter Twenty-three: Reference Documents

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Docker Deployment Documentation:

    https://min.io/docs/minio/container/index.html

MinIO Reverse Proxy Documentation:

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