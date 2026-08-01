# MinIO Access Entry Design: Internal HTTP Communication and External HTTPS Access

Recommended path: 05-Storage/02-MinIO/04-MinIO Access Entry Design: Internal HTTP Communication and External HTTPS Access.md

Tags: #MinIO #ObjectStorage #S3 #HTTPS #HTTP #Nginx #ReverseAgent #UnifiedEntrance #PortPlanning #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the fourth article in the MinIO module, focusing on learning MinIO's access entry design.

Previously completed:

- MinIO object storage basics
- S3 API, Bucket, Object, Prefix
- Single machine single disk deployment
- Single node multi-disk deployment
- 4-node multi-disk distributed cluster deployment
- mc client connection and object upload/download
- Initial state observation after node shutdown

This article begins transitioning from "can access" to "how is the access entry designed".

This article focuses on answering:

    Why can't MinIO directly expose 9000 HTTP to the public internet?
    Why can internal nodes use HTTP?
    Why must external clients use HTTPS?
    How should 9000 and 9001 be planned?
    Should API entry and Console entry be separated?
    Where should the reverse proxy unified entry be placed?
    After Nginx/LB, where should clients, mc, and applications access?
    How to control Console access scope in production environments?
    What should the architecture be before writing Nginx configuration?

This article is not a Nginx configuration practical guide. Nginx specific configurations will be expanded in:

    05-MinIO Reverse Proxy: Nginx Unified Entry, Certificates, and 9000 Port Proxy.md

This article focuses on entry architecture, network boundaries, port boundaries, and security boundary design.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the differences between MinIO internal communication and external access.
2. Understand why HTTP can be used in trusted internal networks.
3. Understand why external clients must use HTTPS.
4. Understand the different responsibilities of 9000 API port and 9001 Console port.
5. Plan MinIO API unified entry.
6. Plan MinIO Console management entry.
7. Understand Nginx/LB's position in MinIO architecture.
8. Understand the significance of separating API domain and Console domain.
9. Master common risks in MinIO entry design.
10. Master network, port, and security check methods before entry design.
11. Prepare for the next Nginx HTTPS reverse proxy practical guide.

---

## III. Experimental Environment Continuation

### 3.1 MinIO Cluster Nodes

This article continues from the previous 4-node MinIO distributed cluster:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc Client / Test Client |
| 10.0.0.46 | minio-entry | Nginx / HTTPS Unified Entry |

---

### 3.2 MinIO Server Image

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Source image:

    minio/minio:RELEASE.2025-04-22T22-12-26Z

---

### 3.3 mc Client Image

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Source image:

    minio/mc:RELEASE.2025-04-16T18-13-26Z

---

### 3.4 Operating System

Default:

    Ubuntu Server 22.04.5 LTS

Optional:

    Rocky Linux 9

---

## IV. MinIO Access Entry Overall Architecture

### 4.1 Current Experiment Direct Access Method

In previous experiments, clients directly accessed a specific MinIO node:

    mc / Browser / App
      |
      | HTTP 9000
      v
    10.0.0.41:9000

Console access:

    Browser
      |
      | HTTP 9001
      v
    10.0.0.41:9001

This method is suitable for experiments but not for production.

Reasons:

    Clients directly perceive backend nodes.
    Clients need manual switching when a node fails.
    HTTP plaintext transmission.
    AccessKey/SecretKey-related requests have security risks.
    Port exposure is inconsistent.
    Console is easily mis-exposed.
    Subsequent certificate, audit, rate limiting, and access control management are difficult to unify.

---

### 4.2 Recommended Production Access Method

Recommended architecture:

    Client / App / mc
      |
      | HTTPS 443
      v
    Nginx / LB Unified Entry
      |
      | HTTP 9000
      v
    MinIO 4-node distributed cluster

Diagram: /think

```
┌──────────────────────────────┐
│ Client / App / mc / SDK       │
└──────────────┬───────────────┘
               │
               │ HTTPS 443
               v
┌──────────────────────────────┐
│ minio-entry / Nginx / LB      │
│ 10.0.0.46                    │
└──────────────┬───────────────┘
               │
               │ HTTP 9000
               v
┌──────────────────────────────┐
│ MinIO Distributed Cluster     │
│ 10.0.0.41:9000               │
│ 10.0.0.42:9000               │
│ 10.0.0.43:9000               │
│ 10.0.0.44:9000               │
└──────────────────────────────┘

Core Principles:

    Internal node communication uses HTTP.
    External client access uses HTTPS.
    Clients only access the unified entry point.
    Backend nodes are not directly exposed to the public internet.
    Console management entry is separately controlled.

---

## V. Responsibilities of Ports 9000 and 9001

### 5.1 Port 9000: S3 API Port

Port 9000 is the MinIO S3 API port.

Main access objects:

    Applications
    mc client
    aws cli
    s3cmd
    Java S3 SDK
    Go S3 SDK
    Python boto3
    Backup synchronization tools
    CI/CD tools

Common operations:

    Create Bucket
    Upload objects
    Download objects
    Delete objects
    List objects
    View object metadata
    Backup synchronization
    Cross-cluster mirror

Example:

    http://10.0.0.41:9000

Production recommendations:

    Do not expose HTTP 9000 directly to the public internet.
    Expose HTTPS 443 through Nginx / LB.
    Proxy to individual MinIO node 9000 in the backend.

---

### 5.2 Port 9001: Web Console Port

Port 9001 is the MinIO Web Console port.

Main access objects:

    Operations personnel
    Administrators
    Storage administrators

Common operations:

    View cluster status
    Manage Bucket
    Manage users
    Manage Policy
    View capacity
    View objects
    View partial monitoring information

Example:

    http://10.0.0.41:9001

Production recommendations:

    Do not expose to the public internet.
    Not recommended to coexist with S3 API in the same public entry point.
    Recommended to use a separate domain name.
    Recommended to restrict source network segments.
    Recommended to allow only VPN, bastion host, or operations network access.
    Not recommended to open Console to regular business users.

---

### 5.3 Port Planning Table

| Port | Purpose | Access Objects | Recommended for Public Exposure |
|---|---|---|---|
| 9000 | S3 API | Applications, SDKs, mc, backup tools | Not recommended to expose HTTP directly |
| 9001 | Web Console | Operations personnel | Not recommended to expose to public internet |
| 443 | HTTPS Unified Entry | Clients, applications, mc | Recommended |
| 80 | HTTP Redirect to HTTPS | Clients | Optional, only for redirection |

---

## VI. Internal HTTP Communication Design

### 6.1 What is Internal HTTP

The internal HTTP referred to in this document means:

    HTTP 9000 used between Nginx / LB and MinIO nodes
    MinIO nodes operate in a trusted internal network
    Backend addresses are 10.0.0.41-10.0.0.44
    Network range is 10.0.0.0/24

Examples:

    http://10.0.0.41:9000
    http://10.0.0.42:9000
    http://10.0.0.43:9000
    http://10.0.0.44:9000

---

### 6.2 Why Internal HTTP Can Be Used

Prerequisites for using HTTP internally:

    MinIO nodes are in a trusted internal network.
    The 10.0.0.0/24 network is not directly exposed to the public internet.
    Backend ports are only accessible to the entry layer and operations network segments.
    There are firewalls, security groups, or network isolation in place.
    External clients do not directly access backend nodes.
    Operations personnel know the entry boundary.

Advantages of internal HTTP:

    Simple configuration.
    Reduces the distribution and maintenance cost of TLS certificates.
    Lowers internal encryption/decryption overhead.
    More intuitive troubleshooting.
    Centralized certificate handling through reverse proxy later.
    MinIO Erasure Coding and high availability capabilities do not depend on internal HTTPS.

---

### 6.3 Limitations of Internal HTTP

Internal HTTP does not imply an absence of security boundaries.

Must satisfy:

    Backend 9000 is not exposed to the public internet.
    Backend 9001 is not exposed to the public internet.
    Only Nginx / LB can access backend 9000.
    Console 9001 is only accessible to operations network segments.
    Server-to-server network is trusted.
    No unknown clients should directly access MinIO nodes.

Incorrect practices:

    Directly mapping 10.0.0.41:9000 to the public internet.
    Opening security group 0.0.0.0/0 to 9000.
    Allowing business to directly configure multiple MinIO node IPs.
    Opening Console 9001 to the public internet.
    No access control in the internal network.

---

### 6.4 Internal HTTP Check Commands

On minio-entry, check backend MinIO nodes:

    curl -I http://10.0.0.41:9000/minio/health/live
    curl -I http://10.0.0.42:9000/minio/health/live
    curl -I http://10.0.0.43:9000/minio/health/live
    curl -I http://10.0.0.44:9000/minio/health/live

Check readiness:

    curl -I http://10.0.0.41:9000/minio/health/ready
    curl -I http://10.0.0.42:9000/minio/health/ready
    curl -I http://10.0.0.43:9000/minio/health/ready
    curl -I http://10.0.0.44:9000/minio/health/ready

Port check:

    nc -vz 10.0.0.41 9000
    nc -vz 10.0.0.42 9000
    nc -vz 10.0.0.43 9000
    nc -vz 10.0.0.44 9000

If no nc: /think
```

apt update
apt install -y netcat-openbsd

---

## Seven. External HTTPS Access Design

### 7.1 Why External Must Use HTTPS

External access must use HTTPS because:

    S3 requests involve AccessKey / SecretKey signing.
    Object data may contain business-sensitive files.
    HTTP plaintext transmission is easily packet-sniffed.
    Man-in-the-middle attacks may tamper requests.
    Object upload/download requires integrity and identity protection.
    Production compliance typically requires HTTPS.
    Subsequent auditing, certificate management, and access control are easier to unify.

---

### 7.2 Recommended API Domain

Recommend planning a dedicated domain for S3 API:

    https://s3.example.com

Experimental environments may use:

    https://s3.minio.local

Or:

    https://minio-s3.example.com

Explanation:

    This domain is for applications, mc, and SDKs to use.
    It should proxy to backend MinIO 9000.
    It should not proxy to Console 9001.
    It should not expose backend node IPs to business.

---

### 7.3 Recommended Console Domain

Recommend planning a dedicated domain for Console:

    https://minio-console.example.com

Experimental environments may use:

    https://console.minio.local

Explanation:

    This domain is only for operations personnel.
    Source IP should be restricted.
    Access may be through VPN, bastion host, or internal network.
    Public internet access is not recommended.
    Console entry should not be handed over to business applications.

---

### 7.4 Benefits of Separating API and Console

Separation benefits:

    Clear responsibilities.
    API faces applications.
    Console faces operations.
    Additional access control can be added to Console.
    Proxy optimization for large file uploads can be done for API.
    Certificates, logs, and auditing can be handled separately.
    Avoid business applications mistaking Console address as S3 Endpoint.
    Reduce security risks from Console exposure.

---

### 7.5 External HTTPS Architecture Diagram

    ┌────────────────────────────┐
    │ App / mc / SDK             │
    └────────────┬───────────────┘
                 │
                 │ https://s3.example.com
                 v
    ┌────────────────────────────┐
    │ Nginx / LB                 │
    │ 443 -> MinIO API 9000      │
    └────────────┬───────────────┘
                 │
                 │ http://10.0.0.41-44:9000
                 v
    ┌────────────────────────────┐
    │ MinIO API                  │
    └────────────────────────────┘


    ┌────────────────────────────┐
    │ Ops User / Admin           │
    └────────────┬───────────────┘
                 │
                 │ https://console.example.com
                 v
    ┌────────────────────────────┐
    │ Nginx / LB                 │
    │ 443 -> MinIO Console 9001  │
    │ allow operations network segment             │
    └────────────┬───────────────┘
                 │
                 │ http://10.0.0.41-44:9001
                 v
    ┌────────────────────────────┐
    │ MinIO Console              │
    └────────────────────────────┘

---

## Eight. Domain and Certificate Planning

### 8.1 Recommended Domain Planning

Production recommendation:

| Domain | Purpose | Backend |
|---|---|---|
| s3.example.com | S3 API | MinIO 9000 |
| minio-console.example.com | Web Console | MinIO 9001 |

Experimental recommendation:

| Domain | Purpose | Backend |
|---|---|---|
| s3.minio.local | S3 API | MinIO 9000 |
| console.minio.local | Web Console | MinIO 9001 |

---

### 8.2 hosts Experimental Configuration

If no internal DNS is available, configure hosts on test clients:

    cat >> /etc/hosts <<'EOF'
    10.0.0.46 s3.minio.local
    10.0.0.46 console.minio.local
    EOF

Verification:

    ping -c 2 s3.minio.local
    ping -c 2 console.minio.local

Windows clients can modify:

    C:\Windows\System32\drivers\etc\hosts

Example:

    10.0.0.46 s3.minio.local
    10.0.0.46 console.minio.local

---

### 8.3 Certificate Planning

Production recommendation:

    Use official trusted certificates.
    Certificate domains must match access domains.
    Certificates should be unified in entry-level Nginx / LB.
    Backend MinIO nodes continue to use HTTP.
    Certificate expiration should be monitored and alarmed.

Certificate file examples:

    /etc/nginx/ssl/s3.minio.local.crt
    /etc/nginx/ssl/s3.minio.local.key
    /etc/nginx/ssl/console.minio.local.crt
    /etc/nginx/ssl/console.minio.local.key

If API and Console use the same wildcard certificate, it can also be:

    *.minio.local

Production environments should use real domain certificates.

---

## Nine. Nginx / LB Position Design

### 9.1 Nginx Entry Node

This document plans: /think

10.0.0.46 minio-entry

Purpose:

    Receive client HTTPS requests.
    Uniformly proxy to backend MinIO nodes.
    Uniformly manage certificates.
    Uniformly manage access logs.
    Can enforce source IP restrictions.
    Can serve as a frontend node for subsequent WAF / LB integration.

---

### 9.2 Nginx to MinIO Backend

API Backend:

    http://10.0.0.41:9000
    http://10.0.0.42:9000
    http://10.0.0.43:9000
    http://10.0.0.44:9000

Console Backend:

    http://10.0.0.41:9001
    http://10.0.0.42:9001
    http://10.0.0.43:9001
    http://10.0.0.44:9001

Notes:

    API and Console are not on the same port.
    Nginx configuration cannot mix usage.
    mc and applications should access API domain names.
    Browser management should access Console domain names.

---

### 9.3 Single Entry vs. Multiple Entries

Optional Scheme 1: Expose only API entry

    https://s3.example.com

Console accessed only via internal node:

    http://10.0.0.41:9001

Suitable for:

    Few administrators.
    Internal operations network reachable.
    No desire for Console to enter public entry.

---

Optional Scheme 2: API and Console both go through Nginx

    https://s3.example.com
    https://minio-console.example.com

Suitable for:

    Want unified certificates.
    Want unified auditing.
    Want to restrict Console source via Nginx.
    Want administrators not to directly access backend nodes.

Recommendation:

    Production environment prefers Scheme 2, but must restrict Console access sources.

---

### 9.4 Nginx High Availability Notes

This document temporarily uses a single-node Nginx:

    10.0.0.46 minio-entry

Sufficient for experimentation.

Production environment should consider:

    Two Nginx instances
    Keepalived VIP
    External LB
    Cloud load balancer
    DNS switching
    Entry monitoring alerts

Diagram:

    Client
      |
      v
    VIP / LB
      |
      +--> nginx-entry01
      |
      +--> nginx-entry02
      |
      v
    MinIO Cluster

Notes:

    MinIO backend is a distributed cluster.
    If only one Nginx node exists at the entry layer, it still has a single point of failure.
    High availability design must consider both entry and storage layers.

---

## Ten. MinIO Startup Parameters and Reverse Proxy Relationship

### 10.1 Why Pay Attention to Server URL

When MinIO is deployed behind a reverse proxy, pay attention to:

    Clients actually access HTTPS domain names.
    MinIO backend sees HTTP requests.
    Console redirects, sharing links, browser redirects may require knowing external access address.

Common related variables:

    MINIO_SERVER_URL
    MINIO_BROWSER_REDIRECT_URL

---

### 10.2 MINIO_SERVER_URL

Recommend setting to API external access address in production:

    MINIO_SERVER_URL=https://s3.example.com

Purpose:

    Inform MinIO of the official S3 API access address.
    Avoid generating incorrect internal addresses.
    Helps maintain consistency in reverse proxy scenarios.

Experimental domain example:

    MINIO_SERVER_URL=https://s3.minio.local

---

### 10.3 MINIO_BROWSER_REDIRECT_URL

Recommend setting to Console external access address:

    MINIO_BROWSER_REDIRECT_URL=https://minio-console.example.com

Experimental domain example:

    MINIO_BROWSER_REDIRECT_URL=https://console.minio.local

Purpose:

    Inform MinIO of the browser access entry for Console.
    Avoid incorrect redirects to internal addresses or wrong ports from API entry.

---

### 10.4 Additional Environment Variables in Subsequent Startup Commands

If using Nginx as a unified entry, recommend adding:

    -e MINIO_SERVER_URL=https://s3.minio.local
    -e MINIO_BROWSER_REDIRECT_URL=https://console.minio.local

Example snippet:

    -e MINIO_ROOT_USER=minioadmin
    -e MINIO_ROOT_PASSWORD='MinioAdmin@123456'
    -e MINIO_SERVER_URL=https://s3.minio.local
    -e MINIO_BROWSER_REDIRECT_URL=https://console.minio.local

Notes:

    All four MinIO nodes should remain consistent.
    If formal domain changes, need to synchronize adjustments.
    Modifying environment variables typically requires rebuilding or updating containers.
    Changing entry domain in production is a change operation.

---

## Eleven. Path-Style vs. Virtual-Host-Style Access

### 11.1 Two S3 Access Styles

Common S3 access methods:

Path-Style:

    https://s3.example.com/bucket-name/object-key

Virtual-Host-Style:

    https://bucket-name.s3.example.com/object-key

---

### 11.2 Experimental Stage Recommends Path-Style

Recommend using Path-Style first in experiments.

Reasons:

    Simple domain planning.
    Simple hosts configuration.
    Low certificate requirements.
    Easier Nginx configuration.
    Easier troubleshooting in initial stages.

Example:

    https://s3.minio.local/app-uploads/hello.txt

---

### 11.3 Production Environment Can Evaluate Virtual-Host-Style

In production, if compatibility with more S3 SDKs or business habits is needed, evaluate Virtual-Host-Style.

Additional considerations:

    Wildcard DNS resolution.
    Wildcard certificate.
    Nginx server_name configuration.
    Bucket name and DNS naming rules.
    Application SDK configuration methods.

Example:

    https://app-uploads.s3.example.com/hello.txt

This document will proceed with Path-Style for experiments to reduce complexity.

---

## Twelve. Entry Access Practical Checks

### 12.1 Check Backend MinIO Cluster

Execute on minio-entry or minio-client:

    curl -I http://10.0.0.41:9000/minio/health/live
    curl -I http://10.0.0.42:9000/minio/health/live
    curl -I http://10.0.0.43:9000/minio/health/live
    curl -I http://10.0.0.44:9000/minio/health/live

Check ready: /think

curl -I http://10.0.0.41:9000/minio/health/ready
curl -I http://10.0.0.42:9000/minio/health/ready
curl -I http://10.0.0.43:9000/minio/health/ready
curl -I http://10.0.0.44:9000/minio/health/ready

---

### 12.2 Check Console Port

    curl -I http://10.0.0.41:9001
    curl -I http://10.0.0.42:9001
    curl -I http://10.0.0.43:9001
    curl -I http://10.0.0.44:9001

Note:

    A responsive Console does not mean it should be open to everyone.
    Access sources should be restricted in production environments.

---

### 12.3 Verify with mc Direct Backend Connection

Create mc configuration directory:

    mkdir -p /data/minio/mc-config

Set alias:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-backend http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Check cluster information:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-backend

Note:

    Ensure the backend cluster is normal before completing the entry proxy configuration.
    Do not troubleshoot Nginx when the backend itself is abnormal.

---

### 12.4 Plan Subsequent HTTPS Verification Commands

After completing Nginx in the next section, mc should be changed to access HTTPS entry:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

If using experimental self-signed certificates or internal certificates, additional CA trust handling may be required.

If using formal trusted certificates, certificate validation should not be skipped.

Production principles:

    Not recommended to use --insecure long-term.
    Not recommended to ignore certificate errors.
    Certificates chain should be correctly installed and trusted.

---

## Thirteen. Firewall and Access Control Design

### 13.1 Backend MinIO Node Access Control

Backend MinIO nodes are recommended to:

    Port 9000: Only allow minio-entry, maintenance network segments, and cluster nodes.
    Port 9001: Only allow maintenance network segments.
    Do not allow public access to port 9000.
    Do not allow public access to port 9001.

Example policy:

| Port | Allowed Sources | Notes |
|---|---|---|
| 9000 | 10.0.0.46, 10.0.0.0/24 internal necessary nodes | S3 API backend access |
| 9001 | Maintenance network segment, VPN, bastion host | Console management |
| 443 | Client sources | HTTPS unified entry |
| 22 | Maintenance management network segment | SSH management |

---

### 13.2 Ubuntu ufw Example

If ufw is enabled, you can refer to:

    ufw allow from 10.0.0.46 to any port 9000 proto tcp
    ufw allow from 10.0.0.0/24 to any port 9000 proto tcp
    ufw allow from 10.0.0.0/24 to any port 9001 proto tcp
    ufw status numbered

If it's an entry node minio-entry:

    ufw allow 443/tcp
    ufw allow 80/tcp
    ufw status numbered

Note:

    Specific rules should be combined with actual environments.
    Experimental environments can be relaxed initially.
    Production environments should tighten sources.

---

### 13.3 Cloud Security Group Approach

If deployed in a cloud environment, security groups are recommended:

Backend MinIO nodes:

    Inbound 9000: Only allow entry layer security group or internal network segments.
    Inbound 9001: Only allow maintenance network segments.
    Inbound 22: Only allow bastion host.
    Do not open 9000 to 0.0.0.0/0.
    Do not open 9001 to 0.0.0.0/0.

Entry node:

    Inbound 443: Allow client access.
    Inbound 80: Optional, for HTTP redirect.
    Outbound 9000: Allow access to backend MinIO nodes.
    Outbound 9001: Allow access to backend Console if proxying Console.

---

## Fourteen. Common Access Entry Errors

### 14.1 Configuring Console Address for Application

Error:

    S3_ENDPOINT=http://10.0.0.41:9001

Correct:

    S3_ENDPOINT=http://10.0.0.41:9000

Production later:

    S3_ENDPOINT=https://s3.example.com

Note:

    9001 is Console.
    9000 is S3 API.

---

### 14.2 Exposing HTTP 9000 Publicly

Error:

    http://InternetIP:9000

Risks:

    Plaintext transmission.
    Credential leakage risk.
    Object data leakage risk.
    Missing unified audit.
    Missing unified entry control.
    Not conducive to multi-node expansion.

Correct:

    https://s3.example.com

---

### 14.3 Publicly Open Console

Error:

    https://console.example.com Open to all public

Risks:

    Management entry exposure.
    Brute force attack risk.
    Misoperation risk.
    User and Policy exposure risk.
    Bucket management risk.

Correct:

    Console should only allow access from maintenance network segment, VPN, and bastion host.
    Use HTTPS.
    Use strong passwords.
    Best to combine with additional access control.

---

### 14.4 Mixing API and Console on Same Domain

Error design:

    https://minio.example.com Proxying both API and Console simultaneously

Issues:

    Complex routing.
    Applications are prone to configuration errors.
    Console security policies are difficult to control separately.
    S3 API and Web Console proxy parameters are not completely the same.
    Troubleshooting becomes difficult later.

Recommendation: /think

https://s3.example.com
https://minio-console.example.com

---

### 14.5 Nginx Proxy After Generating Internal Address

Phenomenon:

    Browser access redirects to http://10.0.0.41:9001
    Console page redirection anomalies
    Shared link displays internal IP
    SDK access shows endpoint inconsistency

Possible Causes:

    MINIO_SERVER_URL not set.
    MINIO_BROWSER_REDIRECT_URL not set.
    Incomplete Nginx forwarding headers.
    Incorrect Host header processing.
    X-Forwarded-Proto not correct.

Resolution Direction:

    Set external access URL.
    Preserve Host header.
    Set X-Real-IP.
    Set X-Forwarded-For.
    Set X-Forwarded-Proto.
    Detailed handling in next Nginx configuration section.

---

## Fifteen, Production Entry Design Checklist

### 15.1 API Entry Check

| Check Item | Requirement | Result |
|---|---|---|
| API Domain | Independent domain, e.g., s3.example.com |  |
| Protocol | HTTPS |  |
| Certificate | Valid trusted certificate |  |
| Backend | MinIO 9000 |  |
| Client | Application, mc, SDK |  |
| 9000 Public Exposure | Not allowed to expose directly |  |
| Large File Upload | Nginx parameters must support |  |
| Access Log | Recommended to enable |  |

---

### 15.2 Console Entry Check

| Check Item | Requirement | Result |
|---|---|---|
| Console Domain | Independent domain, e.g., minio-console.example.com |  |
| Protocol | HTTPS |  |
| Backend | MinIO 9001 |  |
| Source Restriction | Only allow operations network segment |  |
| Root Password | Strong password |  |
| Ordinary Business Access | Not allowed |  |
| Audit | Recommended to retain access logs |  |

---

### 15.3 Backend Access Check

| Check Item | Requirement | Result |
|---|---|---|
| Backend 9000 | Only allow entry layer or internal network access |  |
| Backend 9001 | Only allow operations network access |  |
| Node Communication | Internal HTTP reachable |  |
| Health Check | /minio/health/live and ready normal |  |
| Firewall | Source restricted |  |
| Security Group | Not open 9000/9001 to public |  |

---

### 15.4 MinIO Environment Variable Check

| Variable | Recommended Value | Result |
|---|---|---|
| MINIO_SERVER_URL | https://s3.example.com |  |
| MINIO_BROWSER_REDIRECT_URL | https://minio-console.example.com |  |
| MINIO_ROOT_USER | Strong username |  |
| MINIO_ROOT_PASSWORD | Strong password |  |

---

## Sixteen, Operation: Access Entry Design Pre-Validation Process

### 16.1 Step 1: Confirm Backend Cluster Normal

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-backend

---

### 16.2 Step 2: Confirm 4 Nodes API Health

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check $ip"
      curl -I http://$ip:9000/minio/health/live
      curl -I http://$ip:9000/minio/health/ready
    done

---

### 16.3 Step 3: Confirm Entry Node Can Access Backend

Execute on minio-entry:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check $ip:9000"
      nc -vz $ip 9000
    done

---

### 16.4 Step 4: Confirm Domain Resolution to Entry Node

Execute on client:

    ping -c 2 s3.minio.local
    ping -c 2 console.minio.local

Or:

    getent hosts s3.minio.local
    getent hosts console.minio.local

Expected:

    10.0.0.46 s3.minio.local
    10.0.0.46 console.minio.local

---

### 16.5 Step 5: Confirm Subsequent Proxy Targets

API Backend Targets:

    http://10.0.0.41:9000
    http://10.0.0.42:9000
    http://10.0.0.43:9000
    http://10.0.0.44:9000

Console Backend Targets:

    http://10.0.0.41:9001
    http://10.0.0.42:9001
    http://10.0.0.43:9001
    http://10.0.0.44:9001

---

## Seventeen, Access Entry and Application Configuration Relationship

### 17.1 Application Configuration

Applications should not configure backend node IPs.

Not Recommended:

    S3_ENDPOINT=http://10.0.0.41:9000
    S3_ENDPOINT=http://10.0.0.42:9000

Recommended:

    S3_ENDPOINT=https://s3.example.com

Experiment:

    S3_ENDPOINT=https://s3.minio.local

---

### 17.2 mc Configuration

Before experiment, direct connect to backend:

    alias set minio-backend http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

After entry configuration, change to:

    alias set minio-prod https://s3.example.com <access-key> <secret-key>

Experiment entry:

    alias set minio-lab https://s3.minio.local minioadmin 'MinioAdmin@123456'

Note:

    Not recommended to continue using root user configuration for mc in production.
    Should create regular users and independent AccessKey.

---

### 17.3 SDK Configuration Notes

Common SDK configuration items:

    endpoint
    accessKey
    secretKey
    bucket
    region
    pathStyle
    useSSL

Recommendation: /think

# Endpoint uses HTTPS unified entry point.
useSSL is set to true.
pathStyle can be set to true during the experimental phase.
accessKey uses business-specific user.
secretKey is not written into the code repository.
bucket is named according to business and environment.

---

## Eighteen. Operations and Troubleshooting Path

### 18.1 External Access Failure

Troubleshooting order:

    1. Check if DNS resolves to the entry node.
    2. Check if client can reach entry 443.
    3. Check if Nginx is running.
    4. Check if Nginx certificate is normal.
    5. Check if Nginx upstream backend is reachable.
    6. Check if MinIO 9000 is healthy.
    7. Check if mc alias endpoint is written incorrectly.
    8. Check if AccessKey / SecretKey is correct.
    9. Check if Console port is mistaken for API port.
    10. Check for certificate trust issues.

---

### 18.2 Console Access Abnormality

Troubleshooting order:

    1. Check if Console domain is correct.
    2. Check if access is denied by access control.
    3. Check if Nginx proxies to 9001.
    4. Check if MINIO_BROWSER_REDIRECT_URL is correct.
    5. Check if browser redirect address becomes internal IP.
    6. Check if root user password is correct.
    7. Check if backend MinIO nodes are normal.

---

### 18.3 mc Upload Failure

Troubleshooting order:

    1. Check if mc alias endpoint is API address.
    2. Check if endpoint is HTTPS entry.
    3. Check if certificate is trusted.
    4. Check if accessKey / secretKey is correct.
    5. Check if Bucket exists.
    6. Check if user has write permission.
    7. Check if Nginx limits request body size.
    8. Check if backend MinIO meets write quorum.
    9. Check if disk is full.
    10. Check if backend nodes are offline.

---

### 18.4 Large File Upload Failure

Common causes:

    Nginx client_max_body_size is too small.
    proxy_read_timeout is too short.
    proxy_send_timeout is too short.
    Network interruption.
    Backend MinIO node abnormality.
    Client timeout.
    Certificate or HTTPS interruption.
    Insufficient disk space.

Next, in Nginx configuration, the following should be prioritized:

    client_max_body_size
    proxy_buffering
    proxy_request_buffering
    proxy_connect_timeout
    proxy_send_timeout
    proxy_read_timeout

---

## Nineteen. Advanced SRE Design Principles

### 19.1 Do Not Expose Internal Structure

Business should only know:

    https://s3.example.com

Business should not know:

    10.0.0.41:9000
    10.0.0.42:9000
    10.0.0.43:9000
    10.0.0.44:9000

Reasons:

    Backend nodes may change.
    Node failure should not require manual business switching.
    Backend expansion should not affect business configuration.
    Unified entry point facilitates security, audit, and governance.

---

### 19.2 Entry Layer Handles Security Boundary

Entry layer is responsible for:

    HTTPS certificate
    Access log
    Source restriction
    Request size
    Timeout
    API and Console separation
    Backend load balancing
    Health check
    Subsequent WAF or LB integration

MinIO backend is responsible for:

    Object storage
    S3 API
    Erasure Coding
    Bucket
    Policy
    User authentication
    Data read/write
    Data repair

---

### 19.3 Internal/External Network Responsibilities Separation

Internal:

    Node communication.
    Entry to backend.
    Operations inspection.
    Cluster recovery.
    Monitoring collection.

External:

    Application access.
    User upload/download.
    Business system calls S3 API.

Principles:

    Internal focuses on stability and controllability.
    External focuses on security and uniformity.
    Do not expose internal management ports directly to external.

---

### 19.4 Do Not Break Underlying Runtime

If MinIO is deployed with Docker, do not modify Kubernetes's containerd.

If MinIO is deployed in Kubernetes in the future, also:

    Manage configuration through Helm values.
    Manage entry through Ingress / Service.
    Manage secrets through Secret.
    Do not arbitrarily break containerd for pulling images.
    Use fixed version and trusted repository for images.

---

## Twenty. Interview Answering Approach

If asked in an interview:

    How to design MinIO internal HTTP and external HTTPS?

You can answer:

    I will divide MinIO access into internal communication and external access layers. MinIO nodes between each other and between Nginx and MinIO backend nodes, if they are in a trusted internal network such as the same business internal network or security group, can use HTTP to access the backend 9000 port. This reduces certificate maintenance complexity and internal encryption/decryption overhead, and facilitates troubleshooting.
    However, external client access must use HTTPS. Since MinIO is object storage, S3 requests involve AccessKey, SecretKey signing, and object data transmission. Exposing HTTP 9000 to the public would pose risks of plaintext transmission and credential leakage.
    In production, I will place Nginx or load balancer in front to expose https://s3.example.com uniformly to applications, mc, and SDKs, and proxy to MinIO 4 nodes' 9000 port. Console 9001 will not be directly open to the public, usually accessed via https://minio-console.example.comand limit VPN, bastion host, or operations network segment.
    API and Console should be separated. API faces business applications, and Console faces administrators. This allows separate certificate, access control, log audit, and security policy management.
    Meanwhile, in reverse proxy scenarios, I will pay attention to MINIO_SERVER_URL and MINIO_BROWSER_REDIRECT_URL to avoid Console redirection or link generation with internal IPs. Backend 9000 and 9001 should also be restricted by firewall or security group to prevent direct access bypassing the unified entry point.

---

## Twenty-one. Summary of This Article

This article completes the design of MinIO access entry: /think

1. 9000 is the MinIO S3 API port.
2. 9001 is the MinIO Web Console port.
3. In the experimental environment, you can directly access 9000 and 9001.
4. In production environments, it is not recommended to expose HTTP 9000 directly.
5. In production environments, it is not recommended to expose Console 9001 directly.
6. In internal trusted networks, Nginx / LB can use HTTP to access MinIO backend nodes.
7. External clients must use HTTPS for access.
8. It is recommended to use independent domains for API entry, such as s3.example.com.
9. It is recommended to use independent domains for Console entry, such as minio-console.example.com.
10. Separating API and Console improves permissions, security, auditing, and troubleshooting.
11. Nginx / LB should act as a unified entry point, proxying to 10.0.0.41-10.0.0.44's 9000.
12. Console access should be restricted to maintenance network segments, VPN, or bastion hosts.
13. In reverse proxy scenarios, pay attention to MINIO_SERVER_URL and MINIO_BROWSER_REDIRECT_URL.
14. In experimental phases, Path-Style access is recommended to reduce domain and certificate complexity.
15. Backend nodes 9000 and 9001 should be restricted by firewall or security group rules.
16. The next article will cover Nginx HTTPS unified entry, certificates, and 9000 port proxy practical implementation.

---

## 22. Reference Documents

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Docker Deployment Documentation:

    https://min.io/docs/minio/container/index.html

MinIO Distributed Deployment Documentation:

    https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html

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