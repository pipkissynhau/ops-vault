# RustFS Stage Summary: Positioning and Practical Boundaries of a New Object Storage

Suggested path: 05-Storage/04-RustFS/99-RustFS Stage Summary: Positioning and Practical Boundaries of a New Object Storage.md

Tags: #RustFS #ObjectStorage #S3 #MinioComparison #Docker #DistributiveObjectStorage #Nginx #HTTPS #AccessKey #FaultCheck. #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This document is a stage summary for the RustFS module, used to conclude the entire RustFS learning phase.

Previously completed:

    00-RustFS Directory Index
    01-RustFS Basics: S3-Compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Single-Machine and Cluster Modes
    03-RustFS Single-Machine Deployment Practice: Service Startup, Data Directory, and Access Verification
    04-RustFS Cluster Deployment Practice: Multi-Node, Multi-Disk, and Access Entry Points
    05-RustFS vs. MinIO: Architecture, Deployment, Ecosystem, and Operations Differences
    06-RustFS Client Access: S3 API, mc Tool, and Application Integration
    07-RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxy
    08-RustFS Operations and Troubleshooting: Logs, Capacity, Node Anomalies, and Recovery

This document focuses on summarizing:

    What RustFS is
    The relationship between RustFS and MinIO
    The differences between RustFS and Longhorn, Ceph, NFS
    RustFS deployment modes
    The significance of single-machine deployment for RustFS
    The boundaries of cluster deployment for RustFS
    How clients access RustFS
    How RustFS's security design is implemented
    How to plan the Nginx HTTPS entry for RustFS
    How to perform routine inspections and fault diagnosis for RustFS
    Whether RustFS is suitable for production
    How to evaluate RustFS as a new object storage solution
    What is the real value of learning RustFS for advanced SREs

---

## II. Stage Positioning

RustFS is an S3-compatible object storage system written in Rust.

It can be understood as:

    RustFS is a private S3 object storage service.

Its core positioning is:

    Provide S3 API externally
    Organize data using the Bucket/Object model
    Authenticate via AccessKey/SecretKey
    Support client access via mc, AWS CLI, S3 SDK, etc.
    Used to store unstructured object data such as images, attachments, backup packages, log archives, model files, and AI datasets

RustFS is not:

    Block storage
    File system
    Kubernetes CSI plugin
    Replacement for Longhorn
    Replacement for Ceph RBD
    Replacement for NFS
    Real-time data directory for databases
    Ordinary shared directory

One-sentence summary:

    If an application needs object upload/download interfaces, evaluate RustFS/MinIO/S3.
    If a Pod needs a persistent disk, use Longhorn/Ceph RBD/cloud disk.
    If multiple nodes need shared directories, use NFS/CephFS/NAS.

---

## III. Summary of RustFS Core Models

### 3.1 S3-Compatible Object Storage

RustFS provides S3-compatible interfaces externally.

S3 compatibility means:

    Applications can use S3 SDKs.
    Operations can use mc or AWS CLI.
    Backup programs can use S3 endpoints.
    Data can be synchronized via S3 API or mc mirror during migration.
    Application layers don't directly care about the underlying storage implementation.

But note:

    S3 compatibility doesn't mean all AWS S3 advanced features are identical.
    Basic upload/download passing doesn't equal full production compatibility.
    Production must validate the actual SDK and API capabilities used by the business.

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

Bucket Planning Principles:

    Split by business.
    Split by environment.
    Split by permission boundaries.
    Don't mix all businesses in a single Bucket.
    Don't grant administrator accounts access to all Buckets for all businesses.

---

### 3.3 Object

An Object is the data entity in object storage.

Examples:

    images/avatar-001.png
    resumes/user-1001/resume.pdf
    backups/mysql/full-2026-04-28.sql.gz
    logs/nginx/2026/04/28/access.log.gz
    models/llm/model-v1.bin

Notes:

    The path that appears like a directory in object storage is essentially an Object Key.
    logs/nginx/2026/04/28/access.log.gz is a complete Object Key.
    Prefix is the Key prefix, not a traditional file system directory.

---

### 3.4 Endpoint

An Endpoint is the access entry point for RustFS.

Single-machine mode:

    http://10.0.0.51:9000

Cluster unified entry point:

    http://s3.rustfs.local:9000

Production recommendation:

    https://s3.rustfs.local

Principles:

    Applications should only configure the unified entry point.
    Backend node changes shouldn't affect applications.
    HTTPS is mandatory for external production access.
    Backend node ports shouldn't be exposed directly to the public internet.

---

### 3.5 AccessKey / SecretKey

AccessKey / SecretKey are credentials for accessing RustFS.

Usage principles:

    Administrator keys are only for operations.
    Businesses use independent AccessKeys.
    Different businesses use different keys.
    Different environments use different keys.
    SecretKeys shouldn't be submitted to Git.
    SecretKeys shouldn't be written to public notes.
    SecretKeys shouldn't be printed to logs.
    Keys must be rotatable after leakage.

---

## IV. Relationship Between RustFS and Other Storage Modules

### 4.1 RustFS and MinIO

RustFS and MinIO are the closest.

Commonalities: /think

All are S3-compatible object storage.
All use the Bucket / Object model.
All can use AccessKey / SecretKey.
All can be accessed via mc, AWS CLI, SDK.
All are suitable for images, attachments, backups, log archiving, AI datasets, model files.
All can be provided via Nginx / LB as a unified entry point.
All require HTTPS, permissions, security, logs, capacity, and fault recovery design.

Differences:

| Comparison Item | MinIO | RustFS |
|---|---|---|
| Implementation Language | Go | Rust |
| Maturity | More mature, with more production cases | New solution, requires continuous verification |
| Learning Focus | Mature baseline for private object storage | MinIO's successor and comparison module |
| Production Recommendation | Can be evaluated as a mature S3 solution | Recommend testing, pilot, and canary |
| Operations Experience | More extensive | Needs to be accumulated |
| Ecosystem Tools | More mature | Needs compatibility verification |

One sentence:

    MinIO is the mature baseline.
    RustFS is a new object storage verification target.
    RustFS should not replace production MinIO directly before completing PoC.

---

### 4.2 RustFS and Longhorn

| Comparison Item | RustFS | Longhorn |
|---|---|---|
| Type | Object storage | Kubernetes block storage |
| Access Method | S3 API | PV / PVC / CSI |
| Usage Objects | Applications, SDKs, mc, AWS CLI | Pods, Deployments, StatefulSets |
| Data Unit | Bucket / Object | Volume / Replica |
| Typical Scenarios | Images, attachments, backups, archives | Database data disks, application persistent volumes |
| Suitable as Database Data Directory | Not suitable | Suitable for some scenarios |

One sentence:

    Applications uploading objects should use RustFS / MinIO.
    Pods needing to mount persistent volumes should use Longhorn.

---

### 4.3 RustFS and Ceph RGW

| Comparison Item | RustFS | Ceph RGW |
|---|---|---|
| Type | Independent S3-compatible object storage | Ceph unified storage system's object gateway |
| Underlying Architecture | RustFS's own object storage engine | Ceph RADOS |
| Deployment Complexity | Relatively lightweight | High |
| Operations Complexity | Moderate, requires verification | High |
| Applicable Scenarios | Private S3, replacement verification, lightweight object storage | Existing Ceph clusters needing object interface |
| Learning Focus | S3, deployment, entry point, security, operations | OSD, PG, Pool, CRUSH, RGW |

One sentence:

    RustFS is more lightweight.
    Ceph RGW is more suitable for scenarios with existing Ceph unified storage systems.

---

### 4.4 RustFS and NFS

| Comparison Item | RustFS | NFS |
|---|---|---|
| Type | Object storage | File sharing |
| Access Method | S3 API | Mount shared directory |
| Data Model | Bucket / Object | Files / Directories |
| Typical Scenarios | Attachments, backups, archives | Traditional shared directories, RWX |
| Suitable for Massive Object API | Suitable | Not suitable |
| Suitable for POSIX File Access | Not suitable | Suitable |

One sentence:

    RustFS is an API-based object storage.
    NFS is a shared directory file storage.
    Their usage methods are completely different.

---

## Five, Experiment Environment Summary

### 5.1 Network Planning

RustFS module uses an independent 10.0.0.0/24 experiment plan.

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Node 1 / Single-node node |
| 10.0.0.52 | rustfs-node02 | RustFS Node 2 |
| 10.0.0.53 | rustfs-node03 | RustFS Node 3 |
| 10.0.0.54 | rustfs-node04 | RustFS Node 4 |
| 10.0.0.55 | rustfs-client | mc / AWS CLI client |
| 10.0.0.56 | rustfs-entry | Nginx HTTPS unified entry point |

Avoided:

    10.0.0.10 ops-server
    10.0.0.20 Kubernetes Master
    10.0.0.21 Kubernetes Worker
    10.0.0.22 Kubernetes Worker
    10.0.0.41-10.0.0.46 MinIO experiment nodes

---

### 5.2 Operating System

Default experiment system:

    Ubuntu Server 22.04.5 LTS

Optional supplement:

    Rocky Linux 9

Notes:

    The RustFS module primarily uses Docker deployment.
    Not dependent on Kubernetes.
    Does not modify containerd.
    Does not disrupt existing K8s clusters.
    Consistent with MinIO experiment methods for easy comparison and learning.

---

### 5.3 Image Version

This module experiment fixedly uses:

    rustfs/rustfs:1.0.0-alpha.99

User's Alibaba Cloud image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

Image policy:

    Fixed version.
    Not use latest.
    Pull from official image first.
    Then tag to user's Alibaba Cloud image repository.
    Subsequent experiments use user's image repository.
    Facilitate stable pull under domestic network environment.
    Facilitate experiment reproducibility.

Production reminder:

    This module's fixed version is for learning reproducibility.
    Production version must be re-evaluated based on official Release Notes, security fixes, stability, compatibility, upgrade path, and recovery drills.
    Do not equate experimental version with production recommendation version.

---

## Six, Deployment Mode Summary

### 6.1 SNSD: Single Node Single Disk

SNSD:

    Single Node Single Disk

Experiment node:

    10.0.0.51 rustfs-node01

Data directory:

    /data/rustfs

Suitable for:

    Entry-level learning
    Quick startup
    API verification
    Console verification
    mc verification
    Bucket / Object verification
    Data persistence after container restart

Not suitable for:

    Production high availability
    Node fault tolerance
    Disk fault tolerance
    Core business object storage
    Unique backup target

Conclusion:

Single-node Docker is for entry-level experiments.
Not for production architecture.

---

### 6.2 SNMD: Single Node Multiple Disk

SNMD:

    Single Node Multiple Disk

Example directories:

    /data/rustfs/disk1
    /data/rustfs/disk2
    /data/rustfs/disk3
    /data/rustfs/disk4

Suitable for:

    Single-node multi-disk learning
    Multi-data directory planning
    Understanding single-node multi-disk parameters
    Capacity planning experiments

Not suitable for:

    Node-level high availability
    Production core object storage
    Multi-node disaster recovery

Conclusion:

    Multiple disks do not equal high availability.
    With only one node, node failure makes the service unavailable.

---

### 6.3 MNMD: Multiple Node Multiple Disk

MNMD:

    Multiple Node Multiple Disk

This module plans:

    rustfs-node01 to rustfs-node04
    Each node /data/rustfs0 to /data/rustfs3
    Total 4 nodes x 4 data directories

RUSTFS_VOLUMES example:

    http://rustfs-node{01...04}:9000/data/rustfs{0...3}

Suitable for:

    Cluster deployment learning
    Multi-node object storage verification
    Unified entry point verification
    Node failure simulation
    Recovery simulation
    Pre-production PoC

Production still needs verification:

    S3 API compatibility
    Multipart Upload
    SDK compatibility
    Node failure recovery
    Disk failure recovery
    Capacity expansion
    Performance stress testing
    Permission system
    Log auditing
    Monitoring alerts
    Upgrade rollback

Conclusion:

    MNMD is closer to production form.
    But deployment success does not equal production availability.

---

## Seven, Single-node Deployment Capabilities Summary

Single-node deployment completes the following closed loop:

    Create /data/rustfs
    Set UID 10001 permissions
    Start RustFS with a fixed image
    Map 9000 API port
    Map 9001 Console port
    Configure AccessKey / SecretKey
    Access /health
    Login Console
    Use mc alias set
    Create Bucket
    Upload Object
    Download Object
    sha256sum verification
    Verify data remains after docker restart

Core command review:

    mkdir -p /data/rustfs
    chown -R 10001:10001 /data/rustfs

    docker run -d \
      --name rustfs-single \
      --restart=always \
      -p 9000:9000 \
      -p 9001:9001 \
      -v /data/rustfs:/data \
      -e RUSTFS_ACCESS_KEY="rustfsadmin" \
      -e RUSTFS_SECRET_KEY="RustFSAdmin@123456" \
      -e RUSTFS_CONSOLE_ENABLE=true \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99 \
      --address :9000 \
      --console-enable \
      --access-key "rustfsadmin" \
      --secret-key "RustFSAdmin@123456" \
      /data

Verification:

    docker ps | grep rustfs-single
    docker logs rustfs-single --tail=100
    curl -i http://10.0.0.51:9000/health

Learning value:

    Understand RustFS minimal available path.
    Understand data directory persistence.
    Understand API, Console, mc, Bucket, Object.
    Understand container restart does not equal data loss.
    Understand single-node does not equal high availability.

---

## Eight, Cluster Deployment Capabilities Summary

Cluster deployment completes the following closed loop:

    Plan 4 RustFS nodes
    Configure hostnames
    Configure hosts resolution
    Check node time synchronization
    Prepare 4 data directories per node
    Set UID 10001 permissions
    All nodes use same image version
    All nodes use same RUSTFS_VOLUMES
    All nodes start rustfs-cluster
    Verify /health on each node
    Create Bucket via node01 with mc
    View same Bucket via node02
    Access via Nginx unified entry point
    Simulate stopping node04
    Recover node04
    Verify upload/download

Core startup parameters:

    RUSTFS_VOLUMES="http://rustfs-node{01...04}:9000/data/rustfs{0...3}"

Core startup command:

    docker run -d \
      --name rustfs-cluster \
      --restart=always \
      -p 9000:9000 \
      -p 9001:9001 \
      -v /data/rustfs0:/data/rustfs0 \
      -v /data/rustfs1:/data/rustfs1 \
      -v /data/rustfs2:/data/rustfs2 \
      -v /data/rustfs3:/data/rustfs3 \
      -e RUSTFS_ACCESS_KEY="rustfsadmin" \
      -e RUSTFS_SECRET_KEY="RustFSAdmin@123456" \
      -e RUSTFS_ADDRESS=":9000" \
      -e RUSTFS_CONSOLE_ADDRESS=":9001" \
      -e RUSTFS_CONSOLE_ENABLE="true" \
      -e RUSTFS_VOLUMES="http://rustfs-node{01...04}:9000/data/rustfs{0...3}" \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99 \
      http://rustfs-node{01...04}:9000/data/rustfs{0...3}

Key notes:

Each node startup parameter must be consistent.
Hostname resolution must be correct.
Node time must be synchronized.
Data directory must exist.
Data directory permissions must be correct.
Backend ports must be reachable.
Image version must be consistent.
In production environment, each data directory should correspond to an independent disk.

---

## IX. Unified Entry and HTTPS Summary

### 9.1 Why a Unified Entry is Needed

Unified entry solves:

    Client configuration complexity
    Backend node exposure
    Backend node changes affecting business
    Certificate scattered management
    Log scattering
    Upload size limit inconsistency
    Timeout strategy inconsistency
    Source control inconsistency

Recommended unified entry:

    Client -> HTTPS -> Nginx / LB -> HTTP -> RustFS Backend

---

### 9.2 API and Console Domain Splitting

Recommended:

    S3 API: https://s3.rustfs.local
    Console: https://console.rustfs.local/rustfs/console

Reasons:

    API faces business and clients.
    Console faces operations management.
    Console requires source restriction.
    Separating API and Console makes governance easier.
    Certificates, logs, and security policies are clearer.

---

### 9.3 Nginx Key Parameters

S3 API entry key parameters:

    client_max_body_size 0;
    proxy_request_buffering off;
    proxy_buffering off;
    proxy_connect_timeout 60s;
    proxy_send_timeout 3600s;
    proxy_read_timeout 3600s;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto https;

Reasons:

    Large object uploads require relaxing body limits.
    Disable request buffering to avoid filling Nginx temporary directories.
    Set long timeouts to prevent large file uploads/downloads from being disconnected.
    Host forwarding is critical for S3 signing and Presigned URLs.

---

### 9.4 Internal HTTP vs External HTTPS

Module design:

    Internal node communication: HTTP
    External client access: HTTPS

Applicable premises:

    Internal network is trusted.
    Backend ports are not exposed to the public.
    Firewall restricts sources.
    Management entry restricts sources.
    Certificates are uniformly managed in Nginx / LB.

If crossing untrusted networks:

    Evaluate enabling TLS on backend as well.
    EvaluateLine, VPN, mTLS, or other link encryption solutions.

---

## X. Client Access Capability Summary

This module verifies three types of clients:

    mc
    AWS CLI
    SDK

### 10.1 mc

mc common operations:

    alias set
    ls
    mb
    cp
    stat
    rm
    rb
    mirror

Examples:

    mc alias set rustfs https://s3.rustfs.local <ACCESS_KEY> <SECRET_KEY>
    mc mb rustfs/app-uploads
    mc cp hello.txt rustfs/app-uploads/hello.txt
    mc ls rustfs/app-uploads
    mc cp rustfs/app-uploads/hello.txt ./hello-download.txt

Docker version of mc:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-https

---

### 10.2 AWS CLI

AWS CLI common operations:

    aws --endpoint-url https://s3.rustfs.local s3 ls
    aws --endpoint-url https://s3.rustfs.local s3 mb s3://app-uploads
    aws --endpoint-url https://s3.rustfs.local s3 cp hello.txt s3://app-uploads/hello.txt
    aws --endpoint-url https://s3.rustfs.local s3 presign s3://app-uploads/hello.txt --expires-in 600

Key environment variables:

    AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY
    AWS_DEFAULT_REGION

---

### 10.3 SDK

Must verify actual business SDKs before production:

    Python boto3
    Java AWS SDK
    Node.js AWS SDK
    Go AWS SDK
    Rust S3 SDK
    MinIO SDK

Must confirm:

    endpoint
    region
    credentials
    path-style
    HTTPS
    Presigned URL
    Multipart Upload
    Timeout retry
    Error code handling

Cannot rely solely on mc to determine production readiness.

---

## XI. Path-style vs Virtual-hosted-style Summary

### 11.1 Path-style

Format:

    https://s3.rustfs.local/app-uploads/hello.txt

Structure:

    Endpoint: https://s3.rustfs.local
    Bucket: app-uploads
    Object: hello.txt

Advantages:

    Private deployment is simple.
    No need for wildcard DNS.
    No need for wildcard certificates.
    Nginx configuration is simpler.
    Suitable for experimentation and internal object storage.

---

### 11.2 Virtual-hosted-style

Format:

    https://app-uploads.s3.rustfs.local/hello.txt

Features:

    Closer to AWS S3 access style.
    Requires wildcard DNS.
    Requires wildcard certificates.
    Nginx configuration is more complex.
    SDKs may default to this style.

---

### 11.3 Recommendations

Experimental phase:

    Prioritize Path-style.

Production phase:

    Decide based on business SDK.
    If SDK supports forcePathStyle, prioritize it in private environments.
    If Virtual-hosted-style is required, plan DNS, certificates, and reverse proxy in advance.

---

## 12. Security Design Summary

### 12.1 Administrator Key

Administrator Key is used for:

    Initialization
    Creating Bucket
    Creating AccessKey
    Configuring Permissions
    Operations Management
    Emergency Handling

Should not:

    Be used long-term by business
    Written to application configuration
    Written to CI/CD general variables
    Submitted to Git
    Pasted to public chat
    Shared with multiple businesses

---

### 12.2 Business Key

Business Key should:

    Be independent per business.
    Be independent per environment.
    Be independently authorized per Bucket.
    Only grant necessary actions.
    Be rotated regularly.
    Be auditable.
    Be disableable.

---

### 12.3 Minimum Privileges

Normal upload business:

    s3:GetObject
    s3:PutObject
    s3:ListBucket

Whether to grant:

    s3:DeleteObject

Needs separate evaluation.

Should not grant:

    DeleteBucket
    Administrator permissions
    All Bucket permissions

---

### 12.4 Production Security Baseline

Must achieve:

    External access via HTTPS.
    Console source restriction.
    Backend ports not exposed to public.
    Business key with minimum privileges.
    SecretKey not submitted to Git.
    Certificate expiration monitoring.
    Nginx log retention.
    RustFS log retention.
    AccessKey creation and deletion logged.
    Large object deletion alerts.
    Permission changes require approval.
    Key rotation has process.

---

## 13. Operations and Troubleshooting Summary

### 13.1 Layered Troubleshooting Path

Recommended troubleshooting chain:

    Client / SDK / mc / AWS CLI
        |
        v
    DNS / Endpoint
        |
        v
    HTTPS / Certificate
        |
        v
    Nginx / LB
        |
        v
    RustFS API / Console
        |
        v
    RustFS Container
        |
        v
    Data Directory / Disk
        |
        v
    Node Network / CPU / Memory / Time

---

### 13.2 Daily Inspection Commands

RustFS Node:

    docker ps | grep rustfs
    docker logs rustfs-cluster --tail=100
    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'
    curl -i http://127.0.0.1:9000/health
    df -hT
    du -sh /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3
    timedatectl
    iostat -x 1 10

Nginx Entry:

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

### 13.3 Common Issue Correspondence

| Phenomenon | Priority Check |
|---|---|
| Container startup failure | docker logs, ports, permissions, data directory, RUSTFS_VOLUMES |
| 9000 unreachable | Container, ports, firewall, node network |
| Console unreachable | 9001, Nginx, path, certificate, source restriction |
| AccessDenied | AccessKey, Bucket, Policy, action permissions |
| SignatureDoesNotMatch | SecretKey, time, Endpoint, Host, Path-style |
| 502 | Nginx upstream, backend RustFS health |
| 413 | client_max_body_size |
| 504 | Nginx timeout, backend performance, network, disk I/O |
| Large object upload failure | Nginx buffer, timeout, disk space, Multipart |
| Data write failure | Data disk capacity, directory permissions, node anomaly |
| Certificate error | Certificate chain, domain, expiration, client trust |

---

### 13.4 Capacity Management

Must monitor:

    Data disk usage rate
    System disk usage rate
    Docker log size
    Nginx log size
    Bucket growth trend
    Large object count
    Backup Bucket growth
    Log archive cleanup

Recommended thresholds:

| Usage Rate | Action |
|---|---|
| 70% | Monitor trend |
| 80% | Plan expansion or cleanup |
| 85% | High priority handling |
| 90% | Severe risk |
| 95% | Emergency risk |

---

## 14. Node Abnormality and Recovery Summary

### 14.1 Single Node Abnormality

When a single node is abnormal, confirm:

    Whether it's container or host issue.
    Whether it's port or service issue.
    Whether it's node network or disk issue.
    Whether it affects upload.
    Whether it affects download.
    Whether it affects all Buckets.
    Whether Nginx can still forward to other nodes.

Experimental commands:

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

Focus on: /think

Is the version consistent?  
Are the parameters consistent?  
Is the hostname resolution consistent?  
Is the data directory correct?  
Are the permissions correct?  
Is healing/recovery triggered?  
Has business read/write recovered?

---

### 14.3 Node Replacement

Node replacement principles:

    Keep hostnames as consistent as possible.
    Keep IP or DNS resolution consistent.
    Keep image version consistent.
    Keep startup parameters consistent.
    Keep RUSTFS_VOLUMES consistent.
    Keep AccessKey / SecretKey consistent.
    Keep data directory path consistent.
    Time synchronization is normal.
    Do not replace multiple nodes at the same time.
    Do not manually modify internal data directories.

---

## Fifteen, Production Readiness Assessment

RustFS is suitable for production, but cannot be judged solely by:

    Can it start?
    Can it create a Bucket?
    Can it upload/download?
    Does the official claim high performance?
    Is it compatible with S3?
    Is it newer than MinIO?

Must verify:

    Basic S3 API
    Multipart Upload
    Presigned URL
    SDK compatibility
    Path-style / Virtual-hosted-style
    HTTPS endpoint
    Permission system
    AccessKey management
    Bucket Policy
    Large object upload
    High concurrency for small objects
    Node failure recovery
    Disk failure recovery
    Capacity expansion
    Log auditing
    Prometheus metrics
    Alerting capabilities
    Backup migration
    Version upgrades
    Rollback plan
    Production cases
    Community activity
    License compliance

---

## Sixteen, RustFS Use Case Summary

### 16.1 Recommended Learning and Validation Scenarios

RustFS is suitable for:

    Learning S3 object storage
    Learning new object storage evaluation methods
    Comparative experiments with MinIO
    Validating mc / AWS CLI / SDK
    Validating Nginx HTTPS endpoint
    Validating object storage permission model
    Validating object storage operations and troubleshooting
    Non-core environment pilot
    AI / Data Platform object storage PoC
    Private S3 solution selection testing

---

### 16.2 Considered Pilot Scenarios

Consider:

    Test environment attachment storage
    Non-core business object storage
    Internal tool file upload
    Log archiving experiment
    Backup target experiment
    AI dataset experiment
    Application S3 compatibility verification environment

Requirements:

    Has backup.
    Has monitoring.
    Has recovery plan.
    Has rollback plan.
    Has permission isolation.
    Has capacity alert.

---

### 16.3 Cautionary Production Scenarios

Use with caution for:

    Core business attachment center
    Unique backup target
    Large-scale production data lake
    High-concurrency image storage
    Large model training data foundation
    Financial-grade object storage
    Strong compliance audit scenarios
    Production environment without dedicated operations

A complete PoC must be done before deployment.

---

### 16.4 Not Recommended Scenarios

Not recommended:

    Direct replacement of database data directory.
    Direct replacement of Longhorn.
    Direct replacement of Ceph RBD.
    Direct use as Pod PVC.
    Direct use as traditional shared directory.
    Carrying unique data without backup.
    Providing service without HTTPS.
    Allowing multiple businesses to share without permission governance.
    Production use without monitoring alerts.
    Carrying critical business without recovery drills.
    Replacing production MinIO without PoC.

---

## Seventeen, Production Deployment Acceptance Checklist

### 17.1 Architecture Acceptance

| Check item | Requirements | Result |
|---|---|---|
| Deployment mode | Multi-node multi-disk |  |
| Number of nodes | At least meets current version and business reliability requirements |  |
| Data disk | Independent data disk |  |
| File system | XFS or ext4 |  |
| Hostname resolution | Stable DNS / hosts |  |
| Time synchronization | All nodes synchronized |  |
| Backend port | Not exposed to public internet |  |
| Unified entry | Nginx / LB |  |
| API domain | Independent domain |  |
| Console domain | Independent domain with source restriction |  |

---

### 17.2 Functional Acceptance

| Check item | Requirements | Result |
|---|---|---|
| Bucket creation | Pass |  |
| Object upload | Pass |  |
| Object download | Pass |  |
| Object deletion | Verified by permissions |  |
| ListObjects | Pass |  |
| HeadObject | Pass |  |
| Multipart Upload | Pass |  |
| Presigned URL | Pass |  |
| SDK compatibility | Business actual SDK passes |  |
| Large object upload | Pass |  |
| Small object concurrency | Pass |  |

---

### 17.3 Security Acceptance

| Check item | Requirements | Result |
|---|---|---|
| External HTTPS | Mandatory |  |
| Trusted certificate | Mandatory |  |
| Certificate expiration monitoring | Mandatory |  |
| Administrator key | Not given to business |  |
| Business key | Independent AccessKey |  |
| Minimum permissions | Verified |  |
| Delete permissions | Separate approval |  |
| Console source restriction | Mandatory |  |
| SecretKey management | Not in Git |  |
| Access logs | Retained |  |
| Audit | Traceable |  |

---

### 17.4 Operations Acceptance

| Check item | Requirements | Result |
|---|---|---|
| /health monitoring | Mandatory |  |
| Node monitoring | CPU / memory / disk / network |  |
| Capacity alert | Mandatory |  |
| Nginx 5xx alert | Mandatory |  |
| AccessDenied exception alert | Recommended |  |
| SignatureDoesNotMatch exception alert | Recommended |  |
| Certificate expiration alert | Mandatory |  |
| Large object deletion alert | Mandatory |  |
| Fault record template | Already established |  |
| Recovery drill | Already completed |  |

---

### 17.5 Recovery Acceptance

| Check item | Requirements | Result |
|---|---|---|
| Single node restart | Pass |  |
| Single node stop | Observed impact |  |
| Node recovery | Pass |  |
| Node replacement process | Already rehearsed |  |
| Data verification | Pass |  |
| Backup migration | Already verified |  |
| Rollback plan | Already prepared |  |
| RTO / RPO | Already defined |  |

---

## Eighteen, High-risk Operation Summary

# Strict Operations in Production Environment

    Delete Bucket
    Batch delete Object
    Delete Prefix
    Clear backup Bucket
    Modify AccessKey / SecretKey
    Delete business AccessKey
    Modify Bucket Policy
    Modify Nginx upstream
    Modify certificate
    Modify DNS / hosts
    Modify RUSTFS_VOLUMES
    Modify data directory path
    Format data disk
    Unmount data disk
    Delete /data/rustfs*
    Manually modify RustFS internal data directory
    Simultaneously restart multiple RustFS nodes
    Use nodes of different versions to join cluster
    Turn off certificate validation online
    Provide administrator key to business
    Submit SecretKey to Git

Before execution, must confirm:

    Whether it is production environment.
    Whether there is a maintenance window.
    Whether there is backup.
    Whether there is recovery plan.
    Whether there is rollback plan.
    Whether there is business confirmation.
    Whether on-site logs are retained.
    Whether it has passed dual-person review.

---

## 19. Monitoring and Alert Design Summary

### 19.1 Must Monitor

Must monitor:

    RustFS /health
    RustFS container status
    RustFS API port 9000
    RustFS Console port 9001
    Nginx 80 / 443
    Nginx 502 / 504
    Nginx 413
    Data disk capacity
    System disk capacity
    Docker log size
    Nginx log size
    Node CPU / Memory
    Node disk I/O
    Node network traffic
    Certificate expiration time
    Bucket capacity growth
    Large number of deleted objects
    AccessDenied exceptions increase
    SignatureDoesNotMatch exceptions increase

---

### 19.2 Recommended Monitoring Components

Can plan:

    Prometheus
    Node Exporter
    Nginx Exporter
    Blackbox Exporter
    Grafana
    Alertmanager
    Loki / ELK

Structure:

    Prometheus
      |
      +--> RustFS Metrics
      +--> RustFS /health
      +--> Node Exporter
      +--> Nginx Exporter
      +--> Blackbox Exporter
      +--> Alertmanager

---

### 19.3 Alert Level Classification

| Alert Item | Level |
|---|---|
| RustFS /health is unreachable | Critical |
| Multiple RustFS backends unavailable | Critical |
| Single RustFS backend unavailable | Warning |
| Data disk > 90% | Critical |
| Data disk > 80% | Warning |
| Nginx 502 increase | Critical |
| Nginx 504 increase | Warning / Critical |
| Certificate expires within 15 days | Warning |
| AccessDenied surge | Warning |
| SignatureDoesNotMatch surge | Warning |
| Large number of DeleteObject | Critical |

---

## 20. Fault Record Template

| Item | Content |
|---|---|
| Fault Time |  |
| Discovery Method | Alert / User feedback / Patrol |
| Impact on Business |  |
| Impact on Bucket |  |
| Endpoint |  |
| Whether affect upload | Yes / No |
| Whether affect download | Yes / No |
| Whether affect Console | Yes / No |
| Error Code | 403 / 413 / 502 / 504 / SignatureDoesNotMatch |
| Affected Nodes |  |
| Nginx Log |  |
| RustFS Log |  |
| Disk Capacity |  |
| Node Resources |  |
| Recent Changes |  |
| Preliminary Judgment |  |
| Handling Actions |  |
| Recovery Time |  |
| Root Cause |  |
| Subsequent Improvements |  |

---

## 21. Interview Answering Strategy

If interviewed asked:

    What is RustFS? How do you view its production usage?

Can answer:

# RustFS is a Rust-based S3-compatible object storage system, positioned similarly to MinIO. Its core model is Bucket and Object, and it provides services through the S3 API. Applications can connect via Endpoint, AccessKey, SecretKey, Bucket, and Region parameters, or use mc, AWS CLI, or various S3 SDKs to access it.

I will study RustFS after MinIO. MinIO is a more mature object storage baseline, while RustFS is a new object storage solution, suitable as an alternative, comparison, and PoC (Proof of Concept) scheme for validation.

RustFS is not a block storage system and cannot replace Longhorn. Longhorn provides block storage for Kubernetes Pods via PV/PVC; RustFS offers S3 API for applications, suitable for scenarios like images, attachments, backup packages, log archives, AI datasets, model files, etc. It is not suitable for direct use as a database data directory.

For deployment, I will first do single-node Docker to verify API, Console, Bucket, Object, and data persistence; then do multi-node multi-disk cluster to verify RUSTFS_VOLUMES, hostname resolution, time synchronization, data directories, node failure, and recovery; finally use Nginx as a unified HTTPS entry point, with API and Console split by domain names, external access must be HTTPS, and backend HTTP only placed in trusted internal network.

For security, admin keys are only for operations, business must use independent AccessKey, with minimal permissions by Bucket and action. SecretKey must not be committed to Git, Console must not be exposed publicly, certificates must be monitored for expiration, and Nginx must retain access logs and error logs.

For operations, I will not only check docker ps, but also check /health, Nginx logs, RustFS logs, Bucket access, object upload/download, disk capacity, node resources, certificates, AccessDenied, and SignatureDoesNotMatch. Common issues: 502 check upstream and backend health, 413 check upload size limits, 504 check timeouts and backend performance, SignatureDoesNotMatch check keys, time, Endpoint, Host header, and Path-style.

For production use, my attitude is cautious evaluation. RustFS can be used for learning, verification, and non-core pilot, but if replacing production MinIO or carrying core object data, must fully verify S3 API, Multipart Upload, Presigned URL, SDK compatibility, permission model, large object upload, small object concurrency, node failure recovery, disk failure recovery, monitoring alerts, backup migration, upgrade rollback, and production cases.

---

## Twenty-Two, Capabilities Gained in This Phase

After completing the RustFS phase, the following capabilities should be achieved:

1. Explain what RustFS is.
2. Explain what S3 object storage is.
3. Differentiate RustFS, MinIO, Longhorn, Ceph RGW, and NFS.
4. Understand Bucket, Object, Prefix, Endpoint.
5. Understand AccessKey / SecretKey.
6. Plan a single-node RustFS experiment environment.
7. Plan a multi-node RustFS cluster environment.
8. Deploy RustFS single-node service using Docker.
9. Deploy RustFS multi-node service using Docker.
10. Prepare data directories and permissions.
11. Fix image versions and synchronize to Alibaba Cloud registry.
12. Use mc to create Buckets, upload/download objects.
13. Use AWS CLI to access RustFS.
14. Understand Path-style and Virtual-hosted-style.
15. Generate and verify Presigned URLs.
16. Configure Nginx as a unified entry point.
17. Configure HTTPS.
18. Split API domain and Console domain.
19. Design AccessKey permission isolation.
20. Design Bucket permission isolation.
21. Troubleshoot AccessDenied.
22. Troubleshoot SignatureDoesNotMatch.
23. Troubleshoot Nginx 502 / 413 / 504.
24. Troubleshoot container startup failure.
25. Troubleshoot data directory permission issues.
26. Check capacity, logs, certificates, ports, and health status.
27. Simulate single-node anomalies and recovery.
28. Design production inspection checklist.
29. Design production deployment acceptance checklist.
30. Judge whether RustFS is suitable for a production scenario.

---

## Twenty-Three, Suggestions for Further Learning

After completing the RustFS phase, the following can be continued:

- Compare actual performance between RustFS and MinIO.
- Use boto3 / Java SDK / Node.js SDK for real application integration.
- Validate Multipart Upload.
- Validate Presigned Upload.
- Validate Bucket Policy.
- Validate business-independent AccessKey.
- Validate Prometheus metric integration.
- Validate Grafana Dashboard.
- Validate log auditing.
- Validate node replacement.
- Validate backup migration.
- Validate MinIO -> RustFS data migration.
- Validate RustFS -> MinIO rollback migration.
- Validate HTTPS formal certificate.
- Validate production capacity model.
- Validate high-concurrency small object scenarios.
- Validate large object resumable upload and chunked upload.

---

## Twenty-Four, Summary of This Section

This article completes the summary of the RustFS phase:

1. RustFS is an S3-compatible object storage system written in Rust.  
2. The core model of RustFS is Bucket and Object.  
3. RustFS provides object upload and download capabilities via S3 API.  
4. RustFS is not a block storage system.  
5. RustFS does not replace Longhorn.  
6. RustFS is not suitable for direct use as a database data directory.  
7. RustFS is closest to MinIO.  
8. MinIO is a mature baseline, and RustFS is a new type of validation object.  
9. RustFS can be deployed as a single-node using Docker.  
10. RustFS can be clustered via multi-node multi-disk configuration for validation.  
11. Single-node deployment is suitable for learning but not for production high availability.  
12. Cluster deployment is closer to production form, but deployment success does not guarantee production readiness.  
13. RustFS clients can use mc, AWS CLI, and SDKs.  
14. Business-specific SDKs must be validated before production use.  
15. Path-style is more suitable for private experimentation.  
16. Virtual-hosted-style requires DNS and certificate coordination.  
17. External access must use HTTPS.  
18. Console management entry must restrict source origins.  
19. Administrator keys should not be used long-term by business applications.  
20. Business keys must have minimal permissions.  
21. SecretKey must not be committed to Git.  
22. Nginx entry requires configuration for large object uploads, timeouts, and Host forwarding.  
23. RustFS operations must monitor API, Nginx, logs, capacity, certificates, permissions, and client error messages.  
24. AccessDenied errors should focus on permission troubleshooting.  
25. SignatureDoesNotMatch errors should focus on key, time, Endpoint, Host, and Path-style troubleshooting.  
26. 502 errors should focus on upstream and backend troubleshooting.  
27. 413 errors should focus on upload size limits.  
28. 504 errors should focus on timeouts, performance, and network troubleshooting.  
29. Node recovery should focus on version, parameters, hostname, data directory, and healing.  
30. RustFS must complete full PoC, stress testing, recovery drills, and security acceptance before production use.  
31. At this point, the RustFS phase has completed a full cycle from basic understanding, deployment practice, client integration, security governance, to operations and troubleshooting.  

---

## 25. Reference Documents  

RustFS official website:  

https://rustfs.com/  

RustFS official documentation:  

https://docs.rustfs.com/  

RustFS GitHub:  

https://github.com/rustfs/rustfs  

RustFS Docker installation documentation:  

https://docs.rustfs.com/installation/docker/  

RustFS multi-node multi-disk installation documentation:  

https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html  

RustFS S3 compatibility:  

https://docs.rustfs.com/features/s3-compatibility/  

RustFS Security Checklist:  

https://docs.rustfs.com/installation/checklists/security-checklists.html  

RustFS Access Key Management:  

https://docs.rustfs.com/administration/iam/access-token.html  

RustFS Nginx Reverse Proxy:  

https://docs.rustfs.com/integration/nginx.html  

RustFS TLS Configuration:  

https://docs.rustfs.com/integration/tls-configured.html  

RustFS Logging and Auditing:  

https://docs.rustfs.com/features/logging  

RustFS Node Failure Troubleshooting:  

https://docs.rustfs.com/troubleshooting/node.html  

RustFS mc Client:  

https://docs.rustfs.com/developer/mc.html  

MinIO official documentation:  

https://min.io/docs/minio/linux/index.html  

MinIO mc client documentation:  

https://min.io/docs/minio/linux/reference/minio-mc.html  

AWS S3 API documentation:  

https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html  

AWS S3 Presigned URL documentation:  

https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html  

AWS S3 Virtual Hosting documentation:  

https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html  

AWS CLI S3 documentation:  

https://docs.aws.amazon.com/cli/latest/reference/s3/  

boto3 S3 documentation:  

https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html  

Nginx official documentation:  

https://nginx.org/en/docs/  

Docker official documentation:  

https://docs.docker.com/  

Prometheus official documentation:  

https://prometheus.io/docs/introduction/overview/  

Grafana official documentation:  

https://grafana.com/docs/