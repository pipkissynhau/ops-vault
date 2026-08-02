# Summary of the RustFS Phase: The Positioning and Practical Boundaries of a New Type of Object Storage

Recommended path: 05-Storage/04-RustFS/99-Summary of the RustFS Phase: The Positioning and Practical Boundaries of a New Type of Object Storage.md

Tags: #RustFS #Object Storage #S3 #Comparison with MinIO #Docker #Distributed Object Storage #Nginx #HTTPS #AccessKey #Troubleshooting #Advanced SRE #Production Operations

---

## I. Document Description

This document is a summary of the RustFS module phase, intended to conclude the entire learning process on RustFS.

What has been covered previously includes:

    00-RustFS Directory Index
    01-RustFS Basics: S3-Compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Understanding Single-Node and Cluster Modes
    03-RustFS Single-Node Deployment Practice: Service Startup, Data Directory, and Access Verification
    04-RustFS Cluster Deployment Practice: Multiple Nodes, Multiple Disks, and Access Entrances
    05-RustFS Compared to MinIO: Architectural, Deployment, Ecosystem, and Operations Differences
    06-RustFS Client Access: S3 API, mc Tool, and Application Integration
    07-RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxies
    08-RustFS Operations and Troubleshooting: Logs, Capacity, Node Exceptions, and Recovery

This document focuses on summarizing:

    What RustFS actually is
    The relationship between RustFS and MinIO
    The differences between RustFS and Longhorn, Ceph, and NFS
    RustFS deployment modes
    The significance of single-node RustFS deployment
    The boundaries of RustFS cluster deployment
    How to access RustFS via clients
    How to implement RustFS security design
    How to plan for RustFS Nginx HTTPS entrances
    How to perform RustFS operations inspections and troubleshooting
    Whether RustFS is suitable for production use
    How to evaluate RustFS as a new type of object storage solution
    The true value of learning RustFS for advanced SRE professionals

---

## II. Phase Positioning

RustFS is an S3-compatible object storage system written in Rust.

It can be understood as:

    RustFS is a private S3 object storage service.

Its core positioning is:

    To provide an external S3 API
    To organize data using the Bucket / Object model
    To use AccessKey / SecretKey for authentication
    To support access via clients such as mc, AWS CLI, and S3 SDK
    To be used for storing non-structured object data such as images, attachments, backup packages, log archives, model files, and AI datasets

RustFS is not:

    Block storage
    A file system
    A Kubernetes CSI plugin
    A replacement for Longhorn
    A replacement for Ceph RBD
    A replacement for NFS
    A real-time data directory for databases
    An ordinary shared directory

In one sentence:

    If an application requires object upload and download interfaces, RustFS, MinIO, or S3 can be considered.
    If a Pod needs a persistent disk, Longhorn, Ceph RBD, or cloud disks should be used.
    If multiple nodes need to share directories, NFS, CephFS, or NAS should be utilized.

---

## III. Summary of RustFS's Core Models

### 3.1 S3-Compatible Object Storage

RustFS provides an external S3-compatible interface.

S3 compatibility means:

    Applications can use the S3 SDK.
    Operations personnel can use mc or AWS CLI.
    Backup programs can use the S3 Endpoint.
    During migration, objects can be synchronized via the S3 API or mc mirror.
    The application layer does not need to worry about the underlying storage implementation.

However, it is important to note:

    S3 compatibility does not mean that all advanced AWS S3 features are fully available.
    Just being able to perform basic upload and download operations does not guarantee full production compatibility.
    Production use must verify the capabilities of the SDKs and APIs actually used by the business.

---

### 3.2 Bucket

A Bucket is the top-level space in object storage.

Examples:

    app-uploads
    prod-app-uploads
    prod-backups
    prod-logs-archive
    ai-datasets
    model-files

Principles for planning Buckets:

    Split them according to business needs.
    Split them according to environments.
    Split them based on permission boundaries.
    Do not combine all businesses into one Bucket.
    Do not use an administrator account to grant access to all Buckets for all businesses.

---

### 3.3 Object

An Object is the data entity in object storage.

Examples### 5.2 Operating System

Default experimental system:

    Ubuntu Server 22.04.5 LTS

Optional alternative:

    Rocky Linux 9

Note:

    The RustFS module is primarily deployed using Docker.
    It does not rely on Kubernetes.
    It does not modify containerd.
    It does not disrupt existing Kubernetes clusters.
    This setup is consistent with the MinIO experimental approach, facilitating comparative learning.

---

### 5.3 Image Version

This module experiment consistently uses:

    rustfs/rustfs:1.0.0-alpha.99

User Alibaba Cloud image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

Image strategy:

    Fixed version.
    Avoid using the latest version.
    Pull from the official image first.
    Then tag it to the user's Alibaba Cloud image repository.
    Subsequent experiments will use the user's image repository.
    This ensures stable downloads in domestic network environments and facilitates experiment reproducibility.

Production reminder:

    The fixed version used for this module is intended for learning and reproduction purposes only.
    Production versions must take into account official Release Notes, security fixes, stability, compatibility, upgrade paths, and recovery plan evaluations.
    Do not equate the experimental version with the production-recommended version.

---

## VI. Summary of Deployment Modes

### 6.1 SNSD: Single Node Single Disk

SNSD:

    Single Node Single Disk

Experimental node:

    10.0.0.51 rustfs-node01

Data directory:

    /data/rustfs

Suitable for:

    Beginner learning
    Quick setup
    API verification
    Console verification
    mc usage verification
    Bucket/Object verification
    Verification of data persistence after container restart

Not suitable for:

    High availability in production environments
    Tolerance to node failures
    Tolerance to disk failures
    Core business object storage
    Sole backup target

Conclusion:

    Single-node Docker setup is ideal for beginner experiments.
    It is not suitable for production architectures.

---

### 6.2 SNMD: Single Node Multiple Disks

SNMD:

    Single Node Multiple Disks

Example directories:

    /data/rustfs/disk1
    /data/rustfs/disk2
    /data/rustfs/disk3
    /data/rustfs/disk4

Suitable for:

    Learning with multiple disks on a single node
    Planning multiple data directories
    Understanding parameters related to multiple disks on a single node
    Capacity planning experiments

Not suitable for:

    Node-level high availability
    Core production object storage
    Multi-node disaster recovery

Conclusion:

    Having multiple disks does not necessarily guarantee high availability.
    If there is only one node, service will still be unavailable in the event of a node failure.

---

### 6.3 MNMD: Multiple Nodes Multiple Disks

MNMD:

    Multiple Node Multiple Disks

Planned configuration for this module:

    rustfs-node01 to rustfs-node04
    Each node has data directories /data/rustfs0 to /data/rustfs3
    Total of 4 nodes x 4 data directories

Example for RUSTFS_VOLUMES:

    http://rustfs-node{01...04}:9000/data/rustfs{0...3}

Suitable for:

    Learning about cluster deployment
    Verifying multi-node object storage functionality
    Testing unified access points
    Conducting node failure drills
    Performing recovery tests
    Pre-production proof-of-concept

Production considerations still need to be verified:

    S3 API compatibility
    Multipart Upload support
    SDK compatibility
    Node failure recovery
    Disk failure recovery
    Capacity scaling
    Performance testing
    Permission management system
    Log auditing
    Monitoring and alerts
    Upgrade and rollback procedures

Conclusion:

    MNMD is closer to a production environment setup.
    However, successful deployment does not guarantee production-level availability.

---

## VII. Summary of Single-Node Deployment Capabilities

Single-node deployment has completed the following steps:

    Creating the /data/rustfs directory
    Setting UID 10001 with appropriate permissions
    Starting RustFS using a fixed image
    Mapping API port 9000
    Mapping Console port 9001
    Configuring AccessKey and SecretKey
    Checking the /health endpoint
    Logging in to the Console
    Using mc alias set commands
    Creating Buckets
    Uploading Objects
    Downloading Objects
    Verifying data integrity using sha256sum
    Confirming data persistence after container restart

Core commands:

    mkdir -p /data/rustfs
    chown -R 10001:10001 /data/rustfs

    docker run -dEach node must have consistent startup parameters.  
Host name resolution must be correct.  
Node time must be synchronized.  
The data directory must exist.  
The permissions for the data directory must be appropriate.  
The backend ports must be able to communicate with each other.  
The image versions must be identical.  
In a production environment, each data directory should correspond to an independent disk.  

---

## IX. Unified Access Point and HTTPS Overview  

### 9.1 Why a Unified Access Point is Needed  

A unified access point addresses the following issues:  

    Complex client configurations  
    Exposed backend nodes  
    Changes in backend nodes affecting business operations  
    Scattered management of certificates  
    Dispersed logs  
    Inconsistent upload size limits  
    Varying timeout strategies  
    Lack of unified source control  

It is recommended to use the following access path:  

    Client -> HTTPS -> Nginx / Load Balancer -> HTTP -> RustFS Backend  

---

### 9.2 Separation of API and Console Domain Names  

It is suggested to use the following domain names:  

    S3 API: https://s3.rustfs.local  
    Console: https://console.rustfs.local/rustfs/console  

The reasons are:  

    The API is designed for business operations and client interactions.  
    The console is used for operational management purposes.  
    It is necessary to restrict access to the console.  
    Separating the API and console makes it easier to manage them effectively.  
    This arrangement also helps in organizing certificates, logs, and security policies more clearly.  

---

### 9.3 Key Nginx Configuration Parameters for S3 API Access  

The following parameters are crucial for the S3 API access:  

    `client_max_body_size 0;`  
    `proxy_request_buffering off;`  
    `proxy_buffering off;`  
    `proxy_connect_timeout 60s;`  
    `proxy_send_timeout 3600s;`  
    `proxy_read_timeout 3600s;`  
    `proxy_set_header Host $http_host;`  
    `proxy_set_header X-Forwarded-Proto https;`  

These settings are necessary for reasons such as allowing the upload of large objects, preventing Nginx from running out of temporary storage space, ensuring stable uploads and downloads of large files, and properly setting the `Host` header for S3 signing and Presigned URL generation.  

---

### 9.4 Internal HTTP vs. External HTTPS  

In this design:  

    Internal node communication uses HTTP.  
    External client access uses HTTPS.  

These choices are based on the following assumptions:  

    The internal network is secure.  
    The backend ports are not exposed to the public internet.  
    Source addresses are controlled by firewalls and other security mechanisms.  
    Access to the management interface is also restricted.  
    Certificates are centrally managed through Nginx / Load Balancer.  

If communicating over an untrusted network, it is necessary to consider enabling TLS for the backend as well. Other options such as dedicated lines, VPNs, mTLS, or other encryption solutions should also be evaluated.  

---

## X. Client Access Capabilities Overview  

Three types of clients have been tested in this module:  

    `mc`  
    AWS CLI  
    SDKs  

### 10.1 mc  

Common `mc` operations include:  

    `alias set`  
    `ls`  
    `mb`  
    `cp`  
    `stat`  
    `rm`  
    `rb`  
    `mirror`  

Examples:  

    `mc alias set rustfs https://s3.rustfs.local <ACCESS_KEY> <SECRET_KEY>`  
    `mc mb rustfs/app-uploads`  
    `mc cp hello.txt rustfs/app-uploads/hello.txt`  
    `mc ls rustfs/app-uploads`  
    `mc cp rustfs/app-uploads/hello.txt ./hello-download.txt`  

For the Docker version of `mc`:  

    `docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-https`  

---

### 10.2 AWS CLI  

Common AWS CLI operations include:  

    `aws --endpoint-url https://s3.rustfs.local s3 ls`  
    `aws --endpoint-url https://s3.rustfs.local s3 mb s3://app-uploads`  
    `aws --endpoint-url https://s3.rustfs.local s3 cp hello.txt s3://app-uploads/hello.txt`  
    `aws --endpoint-url https://s3.rustfs    du -sh /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3
    timedatectl
    iostat -x 1 10

Nginx Entrance:

    systemctl status nginx --no-pager
    nginx -t
    ss -lntp | grep ':443'
    curl -k -i https://s3.rustfs.local/health
    tail -100 /var/log/nginx/rustfs-s3-access.log
    tail -100 /var/log/nginx/rustfs-s3-error.log

Client:

    mc ls rustfs-ops
    mc ls --recursive rustfs-ops/prod-app-uploads
    aws --endpoint-url https://s3.rustfs.local s3 ls

---

### 13.3 Common Issues and Corresponding Solutions

| Issue | Priority Check |
|---|---|
| Container fails to start | docker logs, ports, permissions, data directories, RUSTFS_VOLUMES |
| Port 9000 is unreachable | container, ports, firewall, node network |
| Console connection failed | port 9001, Nginx, path, certificate, source restrictions |
| AccessDenied error | AccessKey, Bucket, Policy, permission levels |
| SignatureDoesNotMatch error | SecretKey, time, Endpoint, Host, Path-style |
| 502 error | issues with Nginx upstream or backend RustFS health |
| 413 error | client_max_body_size limit exceeded |
| 504 error | Nginx timeout, backend performance issues, network problems, disk I/O delays |
| Large object upload failure | Nginx buffering issues, timeouts, insufficient disk space, Multipart upload issues |
| Data write failure | insufficient data disk capacity, directory permissions issues, node failures |
| Certificate errors | certificate chain issues, domain name mismatches, certificates expired, client trust issues |

---

### 13.4 Capacity Management

Key areas to monitor:

    Data disk usage rate
    System disk usage rate
    Size of Docker logs
    Size of Nginx logs
    Growth trend of Buckets
    Number of large objects stored
    Growth in backup Buckets
    Log archiving and cleaning procedures

Recommended thresholds:

| Usage Rate | Action Required |
|---|---|
| 70% | Monitor closely |
| 80% | Plan for expansion or cleanup measures |
| 85% | Handle with high priority |
| 90% | Serious risk level |
| 95% | Critical risk level |

---

## Section XIV: Node Failures and Recovery

### 14.1 Single Node Failures

When a single node fails, verify:

    Whether it's a container issue or a host issue.
    Whether the port is faulty or the service is down.
    Whether there are network issues with the node or disk problems.
    Whether uploads or downloads are affected.
    Whether all Buckets are impacted.
    Whether Nginx can still forward requests to other nodes.

Test commands:

    docker stop rustfs-cluster
    curl -k -i https://s3.rustfs.local/health
    mc ls rustfs-ops
    docker start rustfs-cluster

---

### 14.2 Node Recovery

After recovery, check:

    docker ps | grep rustfs
    docker logs rustfs-cluster --tail=200
    curl -i http://10.0.0.54:9000/health
    curl -k -i https://s3.rustfs.local/health
    mc ls rustfs-ops
    mc cp test.txt rustfs-ops/prod-app-uploads/test.txt

Focus on:

    Whether the versions are consistent.
    Whether the configuration parameters match.
    Whether host name resolution is correct.
    Whether the data directory paths are accurate.
    Whether permissions are set correctly.
    Whether healing/recovery mechanisms have been triggered.
    Whether business read/write operations have resumed.

---

### 14.3 Node Replacement

Key principles for node replacement:

    Maintain consistency in host names.
    Ensure IP or DNS resolution remains unchanged.
    Keep the image versions consistent.
    Preserve the same startup parameters.
    Maintain the same RUSTFS_VOLUMES settings.
    Keep AccessKey/SecretKey configurations the same.
    Ensure data directory paths are consistent.
    Verify proper time synchronization.
    Replace multiple nodes at once if possible.
    Avoid manual modifications to internal data directories.

---

## Section XV: Production Availability Assessment

To determine whether RustFS is suitable for production use, consider more than just:

    Whether it can be started up.
    Whether it can create Buckets.
    Whether it can perform uploads and downloads.
    Whether it claims to offer high performance based on official promotions.
    Whether it is compatible with S3.

Essential verification| Recovery Drill | Completed |  |

---

### 17.5 Recovery Acceptance

| Inspection Item | Requirement | Result |
|---|---|---|
| Single-node Restart | Passed |  |
| Single-node Shutdown | Impact observed |  |
| Node Recovery | Passed |  |
| Node Replacement Process | Drilled |  |
| Data Verification | Passed |  |
| Backup Migration | Verified |  |
| Rollback Plan | Prepared |  |
| RTO / RPO | Defined |  |

---

## Section Eighteen: Summary of High-Risk Operations

The following operations must be performed with caution in a production environment:

    Deleting a Bucket
    Batch-deleting Objects
    Deleting a Prefix
    Emptying a backup Bucket
    Modifying an AccessKey / SecretKey
    Deleting a business AccessKey
    Changing the Bucket Policy
    Modifying the Nginx upstream configuration
    Adjusting certificates
    Changing DNS / hosts settings
    Modifying RUSTFS_VOLUMES
    Changing the data directory path
    Formatting a data disk
    Uninstalling a data disk
    Deleting /data/rustfs*
    Manually modifying RustFS internal data directories
    Restarting multiple RustFS nodes simultaneously
    Adding nodes of different versions to the cluster
    Turning off certificate verification before going live
    Providing an administrator key for business use
    Submitting a SecretKey to Git

Before executing any of these operations, it is essential to confirm:

    Whether it is in a production environment.
    Whether there is a maintenance window available.
    Whether backups have been made.
    Whether a recovery plan exists.
    Whether a rollback strategy is in place.
    Whether business approval has been obtained.
    Whether on-site logs have been retained.
    Whether the operations have undergone double-checking.

---

## Section Nineteen: Summary of Monitoring and Alerting Design

### 19.1 Must-be Monitored Metrics

The following metrics must be monitored:

    RustFS /health
    Status of RustFS containers
    RustFS API port 9000
    RustFS Console port 9001
    Nginx ports 80 / 443
    Nginx errors 502 / 504
    Nginx error 413
    Data disk capacity
    System disk capacity
    Size of Docker logs
    Size of Nginx logs
    Node CPU / memory usage
    Node disk I/O performance
    Node network traffic
    Certificate expiration date
    Bucket capacity growth
    Large-scale object deletions
    Increased AccessDenied errors
    Increased SignatureDoesNotMatch errors

---

### 19.2 Recommended Monitoring Components

Components that can be planned for monitoring include:

    Prometheus
    Node Exporter
    Nginx Exporter
    Blackbox Exporter
    Grafana
    Alertmanager
    Loki / ELK

The overall monitoring architecture would look like this:

    Prometheus
      |
      +--> RustFS Metrics
      +--> RustFS /health
      +--> Node Exporter
      +--> Nginx Exporter
      +--> Blackbox Exporter
      +--> Alertmanager

---

### 19.3 Alarm Classification

| Alarm Item | Level |
|---|---|
| RustFS /health failure | Critical |
| Multiple RustFS backends unavailable | Critical |
| Single RustFS backend unavailable | Warning |
    Data disk usage > 90% | Critical |
    Data disk usage > 80% | Warning |
    Increased Nginx error 502 | Critical |
    Increased Nginx error 504 | Warning / Critical |
    Certificate expiration within 15 days | Warning |
    Sudden increase in AccessDenied errors | Warning |
    Sudden increase in SignatureDoesNotMatch errors | Warning |
    Large-scale Object deletions | Critical |

---

## Section Twenty: Fault Record Template

| Item | Details |
|---|---|
| Fault occurrence time |  |
| Discovery method | Alarm / user report / inspection |
    Impact on services |  |
    Impact on Buckets |  |
    Affected endpoints |  |
    Whether upload was affected | Yes / No |
    Whether download was affected | Yes / No |
    Whether the Console was affected | Yes / No |
    Error code | 403 / 413 / 502 / 504 / SignatureDoesNotMatch |
    Affected nodes |  |
    Nginx logs |  |
    RustFS logs |  |
    Disk capacity |  |
    Node resources |  |
    Recent changes |  |
    Preliminary assessment |  |
    Action taken |  |
    Recovery time |  |
    Root cause |  |
    Follow-up improvements |  |

---

## Section Twenty-One: Interview Answer Guidelines

If asked in an interview:

    What isVerify Prometheus metric integration.  
Verify Grafana dashboards.  
Verify log auditing.  
Verify node replacement.  
Verify backup and migration processes.  
Verify data migration from MinIO to RustFS, and vice versa.  
Verify the use of official HTTPS certificates.  
Verify the production capacity model.  
Test high-concurrency scenarios with small objects.  
Ensure support for large-object multipart uploads and segmented uploads.  

---

## Chapter 24: Summary of This Article  

This article provides a summary of the RustFS phase:  

1. RustFS is an object storage system compatible with S3, developed using Rust.  
2. The core components of RustFS are Buckets and Objects.  
3. RustFS offers object upload and download functionality through the S3 API.  
4. RustFS is not designed as a block storage solution.  
5. It does not replace Longhorn.  
6. It is not suitable for use as a direct database data directory.  
7. RustFS is most similar to MinIO in functionality.  
8. MinIO serves as a mature baseline, while RustFS is a new alternative being evaluated.  
9. RustFS can be deployed on a single machine using Docker.  
10. It can also be scaled up in clusters with multiple nodes and disks for testing purposes.  
11. Single-machine deployment is ideal for learning but not for high-availability production use.  
12. Cluster deployment simulates a production environment, but successful setup does not guarantee operational readiness.  
13. RustFS supports clients such as mc, AWS CLI, and SDKs.  
14. Before production use, it is essential to test the actual SDK integration with business requirements.  
15. The Path-style access method is more suitable for private experimentation.  
16. The Virtual-hosted-style requires DNS configuration and certificates.  
17. All external accesses must use HTTPS.  
18. The Console management interface should have source IP restrictions.  
19. Administrator keys should not be shared with business teams for long-term use.  
20. Business-specific keys should have minimal permissions.  
21. SecretKeys must never be stored in Git repositories.  
22. Nginx configuration is necessary to handle large-object uploads, timeouts, and Host routing.  
23. RustFS operations require monitoring of APIs, Nginx, logs, capacity, certificates, permissions, and client errors.  
24. For AccessDenied errors, focus on permission issues.  
25. For SignatureDoesNotMatch errors, check keys, timestamps, endpoints, hosts, and access methods.  
26. For 502 errors, investigate upstream services and backend components.  
27. For 413 errors, check upload size limits.  
28. For 504 errors, examine timeouts, performance issues, and network connectivity.  
29. When recovering from node failures, pay attention to version settings, parameters, hostnames, data directories, and recovery processes.  
30. Before deploying RustFS in production, complete comprehensive proof-of-concept tests, stress tests, recovery drills, and security audits.  
31. To date, the entire development cycle of RustFS, from basic understanding to practical deployment and operational support, has been successfully completed.  

---

## Chapter 25: References  

RustFS Official Website:  
https://rustfs.com/  

RustFS Official Documentation:  
https://docs.rustfs.com/  

RustFS GitHub Page:  
https://github.com/rustfs/rustfs  

RustFS Docker Installation Guide:  
https://docs.rustfs.com/installation/docker/  

RustFS Multi-Node, Multi-Disk Installation Guide:  
https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html  

RustFS S3 Compatibility Information:  
https://docs.rustfs.com/features/s3-compatibility/  

RustFS Security Checklist:  
https://docs.rustfs.com/installation/checklists/security-checklists.html  

RustFS Access Key Management:  
https://docs.rustfs.com/administration/iam/access-token.html  

RustFS Nginx Reverse Proxy Configuration:  
https://docs.rustfs.com/integration/nginx.html  

RustFS TLS Configuration:  
https://docs.rustfs.com/integration/tls-configured.html  

RustFS Logging and Auditing Features:  
https://docs.rustfs.com/features/logging  

RustFS Node Failure Troubleshooting Guide:  
https://docs.rustfs.com/troubleshooting/node.html  

RustFS mc Client Documentation:  
https://docs.rustfs.com/developer/mc.html  

MinIO Official Website:  
https://min.io/docs/minio/linux/index.html  

MinIO mc Client Documentation:  
https://min.io/docs/minio/linux/reference/minio-mc.html  

AWS S3 API Documentation:  
https://docs.aws.amazon.com/AmazonS