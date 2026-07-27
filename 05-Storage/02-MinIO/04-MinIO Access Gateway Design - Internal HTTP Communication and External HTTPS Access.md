# MinIO Access Gateway Design: Internal HTTP Communication and External HTTPS Access

Recommended Path: 05-Storage/02-MinIO/04-MinIO Access Gateway Design: Internal HTTP Communication and External HTTPS Access.md

Tags: #MinIO #Object Storage #S3 #HTTPS #HTTP #Nginx #Reverse Proxy #Unified Gateway #Port Planning #Advanced SRE #Production Operations

---

## I. Document Overview

This document is the fourth part of the MinIO module, focusing on the design of MinIO's access gateway.

Previously covered:

- Basics of MinIO Object Storage
- S3 API, Bucket, Object, Prefix
- Single-machine single-disc deployment
- Single-node multi-disc deployment
- 4-node multi-disk distributed cluster deployment
- mc client connection and object upload/download
- Preliminary observation after node shutdown

This document moves from "being accessible" to "how to design the access gateway."

Key questions this document addresses:

    Why can't MinIO directly expose port 9000 over HTTP to the public network?
    Why can internal nodes use HTTP for communication?
    Why must external client access use HTTPS?
    How should ports 9000 and 9001 be planned?
    Should API access and Console access be separated?
    Where should the reverse proxy unified gateway be placed?
    After setting up Nginx/LB, where should clients, mc, and applications access?
    How to control Console access in a production environment?
    What should the architecture look like before writing Nginx configuration?

This document is not a practical guide for Nginx configuration; specific Nginx settings will be covered in the next article:

    05-MinIO Reverse Proxy: Nginx Unified Gateway, Certificates, and Port 9000 Proxy.md

The focus of this document is on gateway architecture, network boundaries, port boundaries, and security considerations.

---

## II. Learning Objectives

After completing this document, you should be able to:

1. Understand the difference between internal MinIO communication and external access.
2. Comprehend why HTTP can be used within trusted internal networks.
3. Grasp why external client access must use HTTPS.
4. Distinguish between the roles of port 9000 for APIs and port 9001 for the Console.
5. Plan a unified MinIO API gateway.
6. Design a MinIO Console management portal.
7. Understand the role of Nginx/LB in the MinIO architecture.
8. Recognize the significance of separating API and Console domain names.
9. Identify common risks in MinIO access gateway design.
10. Know how to conduct network, port, and security checks before designing an access gateway.
11. Prepare for the practical Nginx HTTPS reverse proxy configuration in the next chapter.

---

## III. Continuing with the Experimental Environment

### 3.1 MinIO Cluster Nodes

This document continues from the previous 4-node MinIO distributed cluster:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc Client / Test Client |
| 10.0.0.46 | minio-entry | Nginx/HTTPS Unified Gateway |

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

## IV. MinIO Access Gateway Overall Architecture

### 4.1 Current Direct Access Method in Experiments

In previous experiments, clients directly accessed a specific MinIO node:

   http://10.0.0.44:9000

---

### 6.2 Why HTTP Can Be Used Internally

Prerequisites for using HTTP internally:

    The MinIO nodes are located within a trusted internal network.
    The IP range 10.0.0.0/24 is not directly exposed to the public internet.
    The backend ports are only accessible from the entry-layer and operations network segments.
    Firewalls, security groups, or network isolation measures are in place.
    External client access does not directly reach the backend nodes.
    Operations personnel are aware of the entry boundaries.

Advantages of using HTTP internally:

    Simple configuration.
    Reduction in the cost of distributing and maintaining TLS certificates.
    Lower internal encryption and decryption overhead.
    Easier troubleshooting.
    Centralized management of certificates through reverse proxies later on.
    MinIO's Erasure Coding and high availability features do not depend on internal HTTPS.

---

### 6.3 Limitations of Internal HTTP

The use of internal HTTP does not mean there are no security boundaries:

    The backend port 9000 must not be exposed to the public internet.
    The backend port 9001 must not be exposed to the public internet.
    Only Nginx or load balancers are allowed to access the backend port 9000.
    The Console should only be accessible from the operations network segment.
    The networks between servers must be trusted.
    Unidentified clients are not allowed direct access to MinIO nodes.

Incorrect practices:

    Directly mapping 10.0.0.41:9000 to the public internet.
    Allowing the security group to allow access from 0.0.0.0/0 to port 9000.
    Allowing services to directly configure multiple MinIO node IPs.
    Exposing the Console port 9001 to the public internet.
    Lacking any access controls within the internal network.

---

### 6.4 Internal HTTP Inspection Commands

To check the backend MinIO nodes on minio-entry:

    curl -I http://10.0.0.41:9000/minio/health/live
    curl -I http://10.0.0.42:9000/minio/health/live
    curl -I http://10.0.0.43:9000/minio/health/live
    curl -I http://10.0.0.44:9000/minio/health/live

To check readiness:

    curl -I http://10.0.0.41:9000/minio/health/ready
    curl -I http://10.0.0.42:9000/minio/health/ready
    curl -I http://10.0.0.43:9000/minio/health/ready
    curl -I http://10.0.0.44:9000/minio/health/ready

To check ports:

    nc -vz 10.0.0.41 9000
    nc -vz 10.0.0.42 9000
    nc -vz 10.0.0.43 9000
    nc -vz 10.0.0.44 9000

If nc is not available:

    Update your package list and install netcat-openbsd.

---

## VII. External HTTPS Access Design

### 7.1 Why External Access Must Use HTTPS

External access must use HTTPS for the following reasons:

    S3 requests involve signing with AccessKey and SecretKey.
    Object data may contain sensitive business information.
    Plain-text HTTP transmissions are easily intercepted.
    Man-in-the-middle attacks can potentially tamper with requests.
    The integrity and authenticity of object uploads and downloads need to be protected.
    Production compliance typically requires HTTPS.
    It facilitates unified management of audit logs, certificates, and access controls.

---

### 7.2 Recommended API Domain Names

It is recommended to dedicate a separate domain name for the S3 API:

    https://s3.example.com

For experimental environments, you can use:

    https://s3.minio.local

or:

    https://minio-s3.example.com

Explanation:

    This domain name should be used by applications, microservices, and SDKs.
    It should proxy requests to the backend MinIO port 9000.
    It should not proxy requests to the Console port 9001.
    The backend node IP addresses should not be exposed to external services.

---

### 7.3 Recommended Console Domain Names

It is recommended to dedicate a separate domain name for the Console:

    https://minIt is possible to implement source IP address restrictions. It can also serve as a front-end node for subsequent connections to WAF/LB systems.

---

### 9.2 Nginx Connecting to the MinIO Backend

API backend:

    http://10.0.0.41:9000
    http://10.0.0.42:9000
    http://10.0.0.43:9000
    http://10.0.0.44:9000

Console backend:

    http://10.0.0.41:9001
    http://10.0.0.42:9001
    http://10.0.0.43:9001
    http://10.0.0.44:9001

Note:

    The API and Console use different ports.
    These settings cannot be mixed when configuring Nginx.
    Both mc and applications should access the API domain name.
    Browser management should use the Console domain name.

---

### 9.3 Single Entry Point or Multiple Entry Points

Option 1: Expose only the API entry point

    https://s3.example.com

The Console can only be accessed through an internal network node:

    http://10.0.0.41:9001

Suitable for:

    Situations with fewer administrators.
    When the internal operations network is accessible.
    When it is not desired to expose the Console through a public entry point.

---

Option 2: Both API and Console go through Nginx

    https://s3.example.com
    https://minio-console.example.com

Suitable for:

    When a unified certificate is required.
    When unified auditing is needed.
    When it is desired to restrict the sources accessing the Console through Nginx.
    When operations personnel are not supposed to directly access the backend nodes.

Recommendation:

    Option 2 is preferred in production environments, but it is essential to restrict the sources accessing the Console.

---

### 9.4 High Availability of Nginx

In this document, a single-node Nginx setup is used for experiments:

    10.0.0.46 minio-entry

This setup is sufficient for testing purposes. In production environments, the following should be considered:

    Two Nginx servers.
    A Keepalived VIP.
    An external Load Balancer.
    Cloud load balancing solutions.
    DNS switching mechanisms.
    Monitoring and alert systems for entry points.

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

Note:

    The MinIO backend is a distributed cluster. Using only one Nginx server at the entry layer still poses a single-point-of-failure risk. High availability design requires considering both the entry layer and the storage layer.

---

## Section X: MinIO Startup Parameters and Their Relationship with Reverse Proxies

### 10.1 Why Pay Attention to the Server URL

When MinIO is deployed behind a reverse proxy, it is important to note that:

    The client actually accesses an HTTPS domain name.
    The MinIO backend receives HTTP requests.
    External access addresses may be required for Console redirections, shared links, or browser redirects.

Common relevant variables:

    MINIO_SERVER_URL
    MINIO_BROWSER_REDIRECT_URL

---

### 10.2 MINIO_SERVER_URL

It is recommended to set this to the external API access address in production environments:

    MINIO_SERVER_URL=https://s3.example.com

Purpose:

    To inform MinIO about the official external S3 API access address.
    To prevent the generation of incorrect internal addresses.
    To ensure consistency in access scenarios involving reverse proxies.

Example domain name for testing:

    MINIO_SERVER_URL=https://s3.minio.local

---

### 10.3 MINIO_BROWSER_REDIRECT_URL

It is recommended to set this to the external Console access address:

    MINIO-browser_redirect_URL=https://minio-console.example.com

Example domain name for testing:

    MINIO_browser.Redirect_URL=https://console.minio.local

Purpose:

    To inform MinIO about the browser access entry point for the Console.
    To prevent incorrect redirects from the API entry point to internal addresses or wrong ports.

---

### 10.4 Additional Environment Variables in Subsequent Startup Commands

If Nginx is used as a unified entry point, it is recommended to add the following environment variables to the MinIO startup commands:

    -e MINIO_SERVER_URL=https://s3.minio.local
    -e MINIO_BROWSER_REDIRECT_URL=https://console.minio.local

Example command snippet:

    -e MINIO_ROOT_USER=minioadmin
```bash
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'
```

If experimental self-signed certificates or internal certificates are used, additional handling of CA trust may be required.

If officially trusted certificates are used, certificate verification should not be skipped.```markdown
10.0.0.46 console.minio.local

---

### 16.5 Step 5: Confirming Subsequent Proxy Targets

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

## Section 17: Relationship Between Access Points and Application Configuration

### 17.1 Application Configuration

Applications should not be configured with backend node IPs.

Not Recommended:

    S3_ENDPOINT=http://10.0.0.41:9000
    S3_endpoint=http://10.0.0.42:9000

Recommended:

    S3_ENDPOINT=https://s3.example.com

Experiment:

    S3endpoint=https://s3.minio.local

---

### 17.2 mc Configuration

Before the experiment, connect directly to the backend:

    alias set minio-backend http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

After setting up the access point, change it to:

    alias set minio-prod https://s3.example.com <access-key> <secret-key>

Experimental Access Point:

    alias set minio-lab https://s3.minio.local minioadmin 'MinioAdmin@123456'

Note:

    In production, it is not recommended to continue using the root user to configure mc for business purposes.
    Ordinary users and separate AccessKeys should be created.

---

### 17.3 Notes on SDK Configuration

Common configuration items for application SDKs include:

    endpoint
    accessKey
    secretKey
    bucket
    region
    pathStyle
    useSSL

Recommendations:

    Use HTTPS as a unified access point for the endpoint.
    Set useSSL to true.
    You can set pathStyle to true during the experimental phase.
    Use a dedicated user account for accessKey.
    Do not store the secretKey in the code repository.
    Name the bucket according to the business and environment.

---

## Section 18: Troubleshooting Paths for Operations and Maintenance

### 18.1 External Access Failure

Troubleshooting sequence:

    1. Check if DNS resolves to the access node.
    2. Verify if the connection from the client to port 443 is established.
    3. Confirm if Nginx is running.
    4. Check if the Nginx certificate is valid.
    5. Ensure that the Nginx upstream backend is accessible.
    6. Verify if MinIO on port 9000 is healthy.
    7. Check if the mc alias endpoint is correctly set.
    8. Confirm if the AccessKey/SecretKey are correct.
    9. Avoid mistaking the Console port for the API port.
    10. Investigate any certificate trust issues.

---

### 18.2 Abnormal Console Access

Troubleshooting sequence:

    1. Verify if the Console domain name is correct.
    2. Check if access control is blocking the request.
    3. Ensure that Nginx is proxying to port 9001.
    4. Confirm if MINIO_BROWSER_REDIRECT_URL is set correctly.
    5. Verify if the browser redirect address results in an internal IP.
    6. Check if the root user password is correct.
    7. Confirm if the backend MinIO node is functioning normally.

---

### 18.3 mc Upload Failure

Troubleshooting sequence:

    1. Verify if the mc alias endpoint is the API address.
    2. Ensure that the endpoint uses HTTPS.
    3. Check if the certificate is trustworthy.
    4. Confirm if the accessKey/secretKey are correct.
    5. Verify if the bucket exists.
    6. Ensure that the user has write permissions.
    7. Check if Nginx limits the request body size.
    8. Confirm if the backend MinIO has reached the required quorum for writing.
    9. Verify if the disk is full.
    10. Check if any backend nodes are offline.

---

### 18.4 Large File Upload Failure

14. During the experimental phase, it is recommended to use the Path-Style access method to reduce the complexity of domain names and certificates.
15. The backend nodes 9000 and 9001 should have their source IP addresses restricted through firewalls or security groups.
16. In the next article, we will delve into the practical aspects of using Nginx as a unified HTTPS gateway, managing certificates, and setting up proxy services for port 9000.

---

## Chapter Twenty-Two: References

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