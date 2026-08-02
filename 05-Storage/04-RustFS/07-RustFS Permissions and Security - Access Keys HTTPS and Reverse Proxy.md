# RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxy

Recommended Path: 05-Storage/04-RustFS/07-RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxy.md

Tags: #RustFS #ObjectStorage #S3 #AccessKeys #PermissionControl #HTTPS #TLS #Nginx #ReverseAgent #Certificate #SecurityBaseline #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the seventh article of the RustFS module, focusing on learning RustFS's permissions, security, access keys, HTTPS, TLS, Nginx reverse proxy, and production security baseline.

Previously completed:

    01-RustFS Basics: S3-Compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Single-Node and Cluster Mode Understanding
    03-RustFS Single-Node Deployment Practice: Service Startup, Data Directory, and Access Verification
    04-RustFS Cluster Deployment Practice: Multi-Node, Multi-Disk, and Access Entry
    05-RustFS vs. MinIO: Architecture, Deployment, Ecosystem, and Operations Differences
    06-RustFS Client Access: S3 API, mc Tool, and Application Integration

This article focuses on solving:

    Why RustFS must have security design
    How to manage AccessKey / SecretKey
    How to differentiate administrator keys and business keys
    How to understand the principle of least privilege
    How to plan Bucket permissions
    Why external access must use HTTPS
    How to balance internal HTTP and external HTTPS
    What's the difference between directly enabling TLS in RustFS and terminating TLS with Nginx
    How to use Nginx to provide a unified HTTPS entry
    How to split S3 API domain and Console management domain
    How to configure certificates
    How to configure mc to use HTTPS endpoint
    How to troubleshoot certificate errors, reverse proxy errors, AccessDenied, and SignatureDoesNotMatch
    How to form a RustFS production security baseline

The experimental strategy used in this article:

    Backend RustFS nodes continue to use internal HTTP
    External access is provided through Nginx unified entry with HTTPS
    S3 API and Console use different domains
    Management entry is restricted by source
    Business access uses independent AccessKey
    Administrator AccessKey is not directly used by business

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the core boundaries of RustFS's security design.
2. Understand the role of AccessKey / SecretKey.
3. Understand the difference between administrator keys and business keys.
4. Create business AccessKey through RustFS Console.
5. Plan different business Buckets.
6. Design minimum privilege policies for business accounts.
7. Understand why SecretKey cannot be submitted to Git.
8. Understand why external access must use HTTPS.
9. Understand the deployment mode of internal HTTP and external HTTPS.
10. Configure HTTPS entry for RustFS S3 API through Nginx.
11. Configure HTTPS entry for RustFS Console through Nginx.
12. Generate experimental self-signed certificates.
13. Replace experimental self-signed certificates with official certificates.
14. Configure HTTP automatic redirect to HTTPS.
15. Configure Nginx upload size, timeout, Host header, and proxy buffering.
16. Use mc to access RustFS through HTTPS.
17. Troubleshoot certificate trust issues.
18. Troubleshoot Nginx 502, 413, 504 errors.
19. Troubleshoot AccessDenied and SignatureDoesNotMatch.
20. Establish a RustFS production security checklist.

---

## III. Core Conclusions First

### 3.1 RustFS Security Is Not Just Setting One Password

Object storage security includes at least:

    AccessKey / SecretKey management
    Administrator account protection
    Business account isolation
    Bucket permission isolation
    Minimum privilege
    HTTPS
    Certificate management
    Reverse proxy security
    Management entry isolation
    Upload size limits
    Timeout control
    Access logs
    Error logs
    Key rotation
    Key leakage handling
    Delete permission control
    Audit and alerts

It is very unprofessional to set one administrator password and have all business share it.

---

### 3.2 Administrator Keys Should Not Be Used Long-Term by Business

Administrator keys are used for:

    Initializing RustFS
    Creating AccessKey
    Creating Bucket
    Configuring permissions
    Operations and maintenance
    Emergency handling

Business keys are used for:

    A business accessing a specific Bucket
    Uploading objects
    Downloading objects
    Listing objects
    Deleting objects

Production principles:

    Administrator keys should not be in business code.
    Administrator keys should not be written to CI/CD general variables.
    Administrator keys should not be in public documentation.
    Administrator keys should not be committed to Git.
    Each business should use independent AccessKey / SecretKey.
    Each business AccessKey should only grant necessary Buckets and necessary actions.

---

### 3.3 External Access Must Use HTTPS

Object storage requests may include:

    AccessKey
    Signature information
    Presigned URL
    Object content
    Upload files
    Download files
    Business attachments
    Backup packages
    Log archives
    Model files

If external access uses HTTP:

    Transmission content may be eavesdropped.
    Signature requests may be intercepted.
    Presigned URL may be leaked.
    Object content may be obtained by a man-in-the-middle.
    It does not meet basic security baseline.

Conclusion:

    Internal network experiments can temporarily use HTTP.
    Production external access must use HTTPS.
    Management entry must use HTTPS.
    Backend internal HTTP can only be placed in trusted networks and isolated by firewall.

---

### 3.4 Recommended to Terminate TLS with Nginx First

RustFS itself supports TLS configuration, but in experiments and small environments, it is recommended to first use:

    Client -> HTTPS -> Nginx / LB -> HTTP -> RustFS backend nodes

Advantages:

    Unified certificate management.
    Unified HTTPS.
    Unified access logs.
    Unified upload size limits.
    Unified timeout configuration.
    Unified source restriction.
    Hide backend nodes.
    Split API and Console into different domains.
    More aligned with common operations and maintenance entry governance methods.

Note: /think

Internal HTTP must only be used within trusted internal networks.
Backend ports 9000 / 9001 should not be exposed to the public internet.
If crossing untrusted networks or across data centers, TLS should be enabled for the backend after assessment.

---

## Four. Experimental Environment

### 4.1 RustFS Cluster Nodes

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Node 1 |
| 10.0.0.52 | rustfs-node02 | RustFS Node 2 |
| 10.0.0.53 | rustfs-node03 | RustFS Node 3 |
| 10.0.0.54 | rustfs-node04 | RustFS Node 4 |
| 10.0.0.55 | rustfs-client | mc / AWS CLI Client |
| 10.0.0.56 | rustfs-entry | Nginx HTTPS Unified Entry |

---

### 4.2 Domain Planning

S3 API Entry:

    s3.rustfs.local

Console Management Entry:

    console.rustfs.local

Hosts Resolution:

    10.0.0.56 s3.rustfs.local
    10.0.0.56 console.rustfs.local

Notes:

    Experimental environment uses /etc/hosts.
    Production environment uses internal DNS or formal DNS.
    API domain and Console domain are recommended to be separated.
    Console domain should restrict source origins.

---

### 4.3 Access Path Planning

Client Access:

    https://s3.rustfs.local

Console Access:

    https://console.rustfs.local/rustfs/console

Backend RustFS API:

    http://10.0.0.51:9000
    http://10.0.0.52:9000
    http://10.0.0.53:9000
    http://10.0.0.54:9000

Backend RustFS Console:

    http://10.0.0.51:9001
    http://10.0.0.52:9001
    http://10.0.0.53:9001
    http://10.0.0.54:9001

---

### 4.4 Certificate Planning

Experimental self-signed certificate directory:

    /etc/nginx/certs/rustfs

Certificate files:

    /etc/nginx/certs/rustfs/rustfs.local.crt
    /etc/nginx/certs/rustfs/rustfs.local.key

Production certificate recommendations:

    Use public CA certificate
    Or enterprise internal CA certificate
    Or certificate issued by unified certificate management system

Production requirements:

    Certificate domain must match.
    Private key permissions must be strictly restricted.
    Certificates must monitor expiration time.
    Certificate renewal must have a process.
    Do not commit private keys to Git.

---

## Five. Security Model Understanding

### 5.1 Root / Admin Account

Root / Admin account is used for:

    System initialization
    Console login
    Creating AccessKey
    Creating or managing Bucket
    Configuring permissions
    Operation and maintenance management

Risks:

    Excessive permissions.
    Once leaked, affects all Buckets.
    If used by business, cannot achieve permission isolation.
    Audit difficulties.

Production recommendations:

    Root / Admin account should only be used for operations.
    Not for general business use.
    Not written to application configuration.
    Not written to CI/CD general variables.
    Only stored in secure channels.
    Regular rotation.
    Immediately disable or replace after leakage.

---

### 5.2 Business AccessKey

Business AccessKey should be applied for:

    Single business
    Single environment
    Specific Bucket
    Specific actions

Example:

| Business | Environment | Bucket | Permissions |
|---|---|---|---|
| app-api | dev | dev-app-uploads | GetObject / PutObject / ListBucket |
| app-api | prod | prod-app-uploads | GetObject / PutObject / ListBucket |
| backup-job | prod | prod-backups | GetObject / PutObject / ListBucket |
| log-archive | prod | prod-logs-archive | PutObject / ListBucket |
| ai-job | test | test-ai-datasets | GetObject / PutObject / ListBucket |

Not recommended:

    All businesses share one AccessKey.
    dev / test / prod share one AccessKey.
    Business account has all Bucket permissions.
    Business account has delete Bucket permissions.
    Business account has administrator permissions.

---

### 5.3 Principle of Least Privilege

Least privilege means:

    Only allow necessary operations.
    Only allow necessary Buckets.
    Only allow necessary Prefixes.
    Default deny for unnecessary actions.

Common actions:

    s3:ListBucket
    s3:GetObject
    s3:PutObject
    s3:DeleteObject
    s3:GetBucketLocation

General upload business:

    ListBucket
    GetObject
    PutObject

Granting DeleteObject requires careful consideration.

Backup tasks:

    PutObject
    ListBucket
    GetObject

Backup cleanup tasks:

    DeleteObject

Delete permissions are recommended to be split separately, do not grant to all businesses by default.

---

## Six. Creating AccessKey via Console

### 6.1 Login to Console

Access:

    https://console.rustfs.local/rustfs/console

In experiments, if HTTPS is not yet configured, you can first use:

    http://10.0.0.51:9001/rustfs/console

Login:

    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456

Production reminder:

    Do not use experimental keys in production.
    Console should not be exposed to the public internet.
    Console should be accessed via VPN / bastion host / operation network segment.
    Console must use HTTPS.

Name: app-api-prod  
Description: Access key for prod app uploads bucket  
Expiration: Set according to current security policy  

After creation, save immediately:  

    AccessKey  
    SecretKey  

Note:  

    SecretKey is typically only displayed once.  
    Save through secure channels after creation.  
    Do not screenshot to group chats.  
    Do not paste to public documentation.  
    Do not commit to Git.  

---

### 6.3 Delete AccessKey  

Operation path:  

    Console  
      -> Access Keys  
      -> Select target AccessKey  
      -> Delete  

Applicable scenarios:  

    Key leakage.  
    Business decommissioning.  
    Employee departure.  
    Key rotation completed.  
    Permission consolidation.  

Before deletion confirmation:  

    Is the business still in use?  
    Is there a new key already in place?  
    Will it affect production uploads/downloads?  
    Has the business party been notified?  

---

### 6.4 AccessKey Record Template  

| Item | Content |  
|---|---|  
| AccessKey name | app-api-prod |  
| Business | app-api |  
| Environment | prod |  
| Bucket | prod-app-uploads |  
| Permission scope | GetObject / PutObject / ListBucket |  
| Allow delete objects | No |  
| Allow delete bucket | No |  
| Creation time |  |  
| Expiration time |  |  
| Responsible person |  |  
| Storage location | Key management system |  
| Rotation cycle |  |  
| Last rotation time |  |  

---

## SevenI don't know.Bucket and Permission Planning  

### 7.1 Bucket Naming Planning  

Recommendations:  

    dev-app-uploads  
    test-app-uploads  
    prod-app-uploads  
    prod-backups  
    prod-logs-archive  
    prod-ai-datasets  
    prod-devops-artifacts  

Not recommended:  

    bucket1  
    test  
    data  
    upload  
    all  
    public  

Naming principles:  

    Reflect environment.  
    Reflect business.  
    Reflect purpose.  
    Lowercase letters, numbers, and hyphens.  
    No Chinese characters.  
    No spaces.  
    No underscores.  

---

### 7.2 Isolate by Environment  

Development environment:  

    dev-app-uploads  

Testing environment:  

    test-app-uploads  

Production environment:  

    prod-app-uploads  

Principles:  

    dev cannot access prod.  
    test cannot access prod.  
    prod keys are not used in test environment.  
    Production bucket permissions are stricter.  
    Production delete permissions require approval.  

---

### 7.3 Isolate by Business  

Business A:  

    prod-app-a-uploads  

Business B:  

    prod-app-b-uploads  

Backup system:  

    prod-backups  

Log archiving:  

    prod-logs-archive  

Principles:  

    A business's key cannot access another business's bucket.  
    Backup keys cannot manage application attachments.  
    Log archiving keys should not delete backup objects.  
    Different business keys rotate independently.  

---

### 7.4 Policy Example: Read-only Permissions  

Read-only policy example:  

    {  
      "Version": "2012-10-17",  
      "Statement": [  
        {  
          "Effect": "Allow",  
          "Action": [  
            "s3:GetBucketLocation",  
            "s3:ListBucket"  
          ],  
          "Resource": [  
            "arn:aws:s3:::prod-app-uploads"  
          ]  
        },  
        {  
          "Effect": "Allow",  
          "Action": [  
            "s3:GetObject"  
          ],  
          "Resource": [  
            "arn:aws:s3:::prod-app-uploads/*"  
          ]  
        }  
      ]  
    }  

Purpose:  

    Static resource reading.  
    File download service.  
    Read-only audit.  
    Migration validation.  

---

### 7.5 Policy Example: Read-write but No Delete  

Read-write policy example:  

    {  
      "Version": "2012-10-17",  
      "Statement": [  
        {  
          "Effect": "Allow",  
          "Action": [  
            "s3:GetBucketLocation",  
            "s3:ListBucket"  
          ],  
          "Resource": [  
            "arn:aws:s3:::prod-app-uploads"  
          ]  
        },  
        {  
          "Effect": "Allow",  
          "Action": [  
            "s3:GetObject",  
            "s3:PutObject"  
          ],  
          "Resource": [  
            "arn:aws:s3:::prod-app-uploads/*"  
          ]  
        }  
      ]  
    }  

Suitable for:  

    Ordinary business file uploads.  
    Backend service file storage.  
    Prevent business from arbitrarily deleting historical objects.  

---

### 7.6 Policy Example: Read-write and Delete Objects  

Read-write and delete objects policy:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::prod-app-uploads"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::prod-app-uploads/*"
      ]
    }
  ]
}

**High-Risk Warning:**

    The DeleteObject permission should be granted with caution.
    Delete permissions should be separated from regular upload permissions.
    Production deletions should have audit and recovery strategies.
    It is not recommended for business users to have default delete bucket permissions.

---

### 7.7 Current Version Permission Function Verification

Due to RustFS's ongoing rapid development, IAM, Policy, and AccessKey behaviors may differ across versions.

Before production use, must verify:

    Whether the current version supports required Policy.
    Whether Policy syntax is fully compatible with S3.
    Whether AccessKey can be bound to specified permissions.
    Whether read-only policies take effect.
    Whether read-write policies take effect.
    Whether delete permissions can be independently controlled.
    Whether bucket-level permissions meet expectations.
    Whether prefix-level permissions meet expectations.
    Whether console and API permission behaviors are consistent.

Verification Principles:

    Do not test only with administrator accounts.
    Must test with business accounts.
    Must verify failed privilege escalation.
    Must verify whether delete permissions are properly restricted.

---

## EightI don't know.Practical Operation 1: Creating a Business Bucket

### 8.1 Configuring mc Administrator Alias

Execute in rustfs-client:

    mkdir -p /data/rustfs-security/mc-config

Configure administrator alias:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-admin http://s3.rustfs.local:9000 rustfsadmin 'RustFSAdmin@123456'

Check:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-admin

---

### 8.2 Creating Production Simulation Bucket

Create:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-admin/prod-app-uploads

Create backup bucket:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-admin/prod-backups

Create log archive bucket:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-admin/prod-logs-archive

Check:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-admin

---

### 8.3 Uploading Test Objects

Prepare files:

    mkdir -p /tmp/rustfs-security-test
    cd /tmp/rustfs-security-test

    echo "security test object" > hello.txt
    echo "backup data" > backup.sql
    echo "nginx access log" > access.log

Upload:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      -v /tmp/rustfs-security-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /test/hello.txt rustfs-admin/prod-app-uploads/hello.txt

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  -v /tmp/rustfs-security-test:/test \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp /test/backup.sql rustfs-admin/prod-backups/backup.sql

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  -v /tmp/rustfs-security-test:/test \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp /test/access.log rustfs-admin/prod-logs-archive/access.log

View:

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls --recursive rustfs-admin/prod-app-uploads

---

## IX. Practical Exercise 2: Business AccessKey Verification

### 9.1 Create Business AccessKey

Create through Console:

Name: app-api-prod  
Description: prod app uploads read write access  
Expiration: Set according to experiment or security policy

Record:

APP_ACCESS_KEY=<Actual Business AccessKey>  
APP_SECRET_KEY=<Actual Business SecretKey>

Security Reminder:

Do not write real SecretKey into notes.  
Do not paste real SecretKey to public chat.  
Do not submit real SecretKey to Git.

---

### 9.2 Configure Business Alias

Execute in rustfs-client:

export APP_ACCESS_KEY="<Actual Business AccessKey>"  
export APP_SECRET_KEY="<Actual Business SecretKey>"

Configure alias:

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  alias set rustfs-app http://s3.rustfs.local:9000 "${APP_ACCESS_KEY}" "${APP_SECRET_KEY}"

View alias:

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  alias list

---

### 9.3 Verify Business Account Can Access Target Bucket

Execute:

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-app/prod-app-uploads

Upload test:

echo "upload by app key" > /tmp/app-key-upload.txt

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  -v /tmp:/tmpdata \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp /tmpdata/app-key-upload.txt rustfs-app/prod-app-uploads/app-key-upload.txt

Download test:

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  -v /tmp:/tmpdata \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp rustfs-app/prod-app-uploads/app-key-upload.txt /tmpdata/app-key-download.txt

Verification:

cat /tmp/app-key-download.txt

---

### 9.4 Verify Business Account Should Not Access Other Buckets

Attempt to access backup bucket:

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-app/prod-backups

Expected:

If permissions are configured correctly, should return AccessDenied or permission error.

Attempt to access log archive bucket: /think

docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-app/prod-logs-archive

Expected:

    Business accounts should not access unrelated Buckets.

---

### 9.5 Verifying Delete Permissions

If business accounts should not delete objects, execute:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm rustfs-app/prod-app-uploads/app-key-upload.txt

Expected:

    If DeleteObject is not authorized, it should return AccessDenied.

If deletion permissions are required, confirm:

    Whether object deletion has audit logs.
    Whether there is a trash bin or version control.
    Whether there is a backup.
    Whether it is confirmed by the business owner.
    Whether bulk deletion is allowed.

---

## TenI don't know.HTTPS Design Plan

### 10.1 Plan One: RustFS Directly Enables TLS

RustFS supports providing HTTPS through TLS-related configuration.

Typical approach:

    Prepare rustfs_cert.pem
    Prepare rustfs_key.pem
    Mount certificate directory
    Set RUSTFS_TLS_PATH
    Restart RustFS

Advantages:

    RustFS process itself provides HTTPS.
    Backend transmission can also be encrypted.
    Reduces one layer of reverse proxy.

Disadvantages:

    Certificate management across multiple nodes is complex.
    Certificate updates require handling each node.
    Console and API entry governance is less flexible than Nginx.
    Upload size, timeout, access logs, source control require additional design.
    If there is still a LB in front, a unified entry governance is still needed.

Suitable for:

    Internal requirements also demand full-chain TLS.
    Has a unified certificate distribution capability.
    Has mature configuration management capability.
    Has cross-untrusted network communication needs.

---

### 10.2 Plan Two: Nginx Terminates TLS

Recommended for experiments and small environments:

    Client
      |
      | HTTPS
      v
    Nginx / LB
      |
      | HTTP Internal Network
      v
    RustFS Backend

Advantages:

    Unified certificate management.
    Unified access logs.
    Unified error logs.
    Unified source restriction.
    Unified upload size limits.
    Unified timeouts.
    Console and API can be split by domain.
    Backend nodes are hidden.
    Easy to switch backend nodes later.

Risks:

    HTTP from Nginx to backend.
    Must ensure the backend network is trusted.
    Must restrict the source of backend ports.
    If cross-datacenter or untrusted network, TLS must also be enabled on the backend.

---

### 10.3 Plan Adopted in This Document

This document adopts:

    External HTTPS: Nginx terminates TLS
    Internal HTTP: Nginx forwards to RustFS backend 9000 / 9001
    API domain: s3.rustfs.local
    Console domain: console.rustfs.local

Experimental access:

    https://s3.rustfs.local
    https://console.rustfs.local/rustfs/console

---

## ElevenI don't know.Practical Three: Prepare Experimental Self-Signed Certificates

### 11.1 Create Certificate Directory

Execute on rustfs-entry:

    mkdir -p /etc/nginx/certs/rustfs
    chmod 700 /etc/nginx/certs/rustfs

---

### 11.2 Generate Self-Signed Certificate Configuration

Create OpenSSL configuration:

    cat > /etc/nginx/certs/rustfs/rustfs-local-openssl.cnf <<'EOF'
    [req]
    default_bits = 2048
    prompt = no
    default_md = sha256
    req_extensions = req_ext
    distinguished_name = dn

    [dn]
    C = CN
    ST = Gansu
    L = Lab
    O = RustFS Lab
    OU = SRE
    CN = s3.rustfs.local

    [req_ext]
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = s3.rustfs.local
    DNS.2 = console.rustfs.local
    DNS.3 = rustfs-entry
    IP.1 = 10.0.0.56
    EOF

---

### 11.3 Generate Certificate and Private Key

Execute:

    openssl req -x509 -nodes -days 365 \
      -newkey rsa:2048 \
      -keyout /etc/nginx/certs/rustfs/rustfs.local.key \
      -out /etc/nginx/certs/rustfs/rustfs.local.crt \
      -config /etc/nginx/certs/rustfs/rustfs-local-openssl.cnf \
      -extensions req_ext

Set permissions:

    chmod 600 /etc/nginx/certs/rustfs/rustfs.local.key
    chmod 644 /etc/nginx/certs/rustfs/rustfs.local.crt

View:

    ls -l /etc/nginx/certs/rustfs

---

### 11.4 View Certificate Information

Execute:

    openssl x509 -in /etc/nginx/certs/rustfs/rustfs.local.crt -noout -text | grep -E "Subject:|DNS:|IP Address" -A2

Confirm contains:

    s3.rustfs.local
    console.rustfs.local
    10.0.0.56

---

### 11.5 Production Certificate Replacement Notes

In production, replace with official certificates: /think

/etc/nginx/certs/rustfs/fullchain.pem
/etc/nginx/certs/rustfs/privkey.pem

Example:

    ssl_certificate     /etc/nginx/certs/rustfs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/rustfs/privkey.pem;

Production Requirements:

    Private key permissions 600.
    Owner user root.
    Complete certificate chain.
    Domain name match.
    Renewal process in place.
    Expiration alert system.

---

## Twelve、Practical Step Four: Configure Nginx HTTPS API Entry

### 12.1 Install Nginx

Execute on rustfs-entry:

    apt update
    apt install -y nginx

Start:

    systemctl enable --now nginx

Check:

    nginx -v
    systemctl status nginx --no-pager

---

### 12.2 Create S3 API Reverse Proxy Configuration

Create configuration file:

    cat > /etc/nginx/conf.d/rustfs-s3-https.conf <<'EOF'
    upstream rustfs_s3_backend {
        least_conn;
        server 10.0.0.51:9000 max_fails=3 fail_timeout=30s;
        server 10.0.0.52:9000 max_fails=3 fail_timeout=30s;
        server 10.0.0.53:9000 max_fails=3 fail_timeout=30s;
        server 10.0.0.54:9000 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name s3.rustfs.local;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name s3.rustfs.local;

        ssl_certificate     /etc/nginx/certs/rustfs/rustfs.local.crt;
        ssl_certificate_key /etc/nginx/certs/rustfs/rustfs.local.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;

        client_max_body_size 0;
        client_body_timeout 3600s;

        proxy_request_buffering off;
        proxy_buffering off;

        proxy_connect_timeout 60s;
        proxy_send_timeout 3600s;
        proxy_read_timeout 3600s;
        send_timeout 3600s;

        access_log /var/log/nginx/rustfs-s3-access.log;
        error_log  /var/log/nginx/rustfs-s3-error.log;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;

            proxy_http_version 1.1;
            proxy_set_header Connection "";

            proxy_pass http://rustfs_s3_backend;
        }
    }
    EOF

---

### 12.3 Check and Reload Nginx

Execute:

    nginx -t
    systemctl reload nginx

Check ports:

    ss -lntp | grep ':443'
    ss -lntp | grep ':80'

---

### 12.4 Configure Client hosts

Execute on rustfs-client:

    cat >> /etc/hosts <<'EOF'
    10.0.0.56 s3.rustfs.local
    10.0.0.56 console.rustfs.local
    EOF

Check:

    getent hosts s3.rustfs.local
    getent hosts console.rustfs.local

---

### 12.5 Test HTTPS Health Check

Since it's a self-signed certificate, first test with -k:

    curl -k -i https://s3.rustfs.local/health

Expected:

    HTTP/2 200
    Or HTTP/1.1 200 OK

If failed, check:

    nginx -t
    systemctl status nginx --no-pager
    tail -100 /var/log/nginx/rustfs-s3-error.log
    curl -i http://10.0.0.51:9000/health

---

## Thirteen、Practical Step Five: Configure Nginx HTTPS Console Entry

### 13.1 Console Entry Design

Console uses a separate domain:

    console.rustfs.local

Backend port:

    9001

Production Principles:

    Console should not be exposed to the public internet.
    Console should restrict source IP addresses.
    Console must use HTTPS.
    Console and S3 API should not share the same domain path, reducing proxy complexity.
    Management access should be through VPN / bastion host / operations network segment.

---

### 13.2 Create Console Reverse Proxy Configuration

In rustfs-entry execute:

    cat > /etc/nginx/conf.d/rustfs-console-https.conf <<'EOF'
    upstream rustfs_console_backend {
        ip_hash;
        server 10.0.0.51:9001 max_fails=3 fail_timeout=30s;
        server 10.0.0.52:9001 max_fails=3 fail_timeout=30s;
        server 10.0.0.53:9001 max_fails=3 fail_timeout=30s;
        server 10.0.0.54:9001 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name console.rustfs.local;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name console.rustfs.local;

        ssl_certificate     /etc/nginx/certs/rustfs/rustfs.local.crt;
        ssl_certificate_key /etc/nginx/certs/rustfs/rustfs.local.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;

        client_max_body_size 200m;

        proxy_connect_timeout 60s;
        proxy_send_timeout 3600s;
        proxy_read_timeout 3600s;
        send_timeout 3600s;

        access_log /var/log/nginx/rustfs-console-access.log;
        error_log  /var/log/nginx/rustfs-console-error.log;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            proxy_pass http://rustfs_console_backend;
        }
    }
    EOF

---

### 13.3 Check and Reload Nginx

Execute:

    nginx -t
    systemctl reload nginx

Check logs:

    tail -50 /var/log/nginx/rustfs-console-error.log

---

### 13.4 Test Console Access

Browser access:

    https://console.rustfs.local/rustfs/console

If using a self-signed certificate:

    The browser will prompt that the certificate is untrusted.
    In experimental environments, you can manually proceed.
    In production environments, a trusted certificate or enterprise CA must be used.

Login:

    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456

---

### 13.5 Restrict Console Source IP

Production recommendation: only allow access from the operations network segment.

Example:

    location / {
        allow 10.0.0.0/24;
        deny all;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_pass http://rustfs_console_backend;
    }

Notes:

    10.0.0.0/24 is the current experimental network segment.
    In production, replace with bastion host, VPN, or maintenance exit IP.
    It is not recommended to expose the Console to all public sources.

---

## Fourteen、Practical Six: Configure mc to Access RustFS via HTTPS

### 14.1 Self-Signed Certificate Access Method

Since the experiment uses a self-signed certificate, mc may not trust the certificate.

You can first test with --insecure.

Configure alias:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set rustfs-https https://s3.rustfs.local rustfsadmin 'RustFSAdmin@123456'

Check:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-https

### 14.2 Uploading Objects via HTTPS

Create file:

    echo "upload through rustfs https entry" > /tmp/rustfs-https-test.txt

Upload:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /tmpdata/rustfs-https-test.txt rustfs-https/prod-app-uploads/rustfs-https-test.txt

Check:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-https/prod-app-uploads

---

### 14.3 Download and Verify

Download:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp rustfs-https/prod-app-uploads/rustfs-https-test.txt /tmpdata/rustfs-https-download.txt

Check:

    cat /tmp/rustfs-https-download.txt

Verify:

    sha256sum /tmp/rustfs-https-test.txt /tmp/rustfs-https-download.txt

---

### 14.4 Do Not Use --insecure Long-Term in Production

--insecure is suitable for:

    Experimenting with self-signed certificates.
    Temporary troubleshooting.
    Non-production verification.

Production should:

    Use trusted certificates.
    Client trusts CA.
    Do not disable certificate verification long-term.
    Certificate domain matches the Endpoint exactly.

---

## FifteenI don't know.Practical Exercise Seven: Let Clients Trust Self-Signed Certificates

### 15.1 Trust Certificate on Ubuntu

Execute on rustfs-client:

    mkdir -p /usr/local/share/ca-certificates/rustfs

Copy certificate from rustfs-entry:

    scp root@10.0.0.56:/etc/nginx/certs/rustfs/rustfs.local.crt \
      /usr/local/share/ca-certificates/rustfs/rustfs.local.crt

Update CA:

    update-ca-certificates

Verify:

    curl -i https://s3.rustfs.local/health

If -k is no longer needed, it means the system now trusts the certificate.

---

### 15.2 Reconfigure mc Without Using --insecure

Try:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-https-trusted https://s3.rustfs.local rustfsadmin 'RustFSAdmin@123456'

Note:

    If mc runs in a container, the container may not trust the host's CA by default.
    Need to mount the CA certificate into the container or build an mc image with enterprise CA.
    Production recommends using public trusted certificates or enterprise unified CA images.

---

### 15.3 Mounting CA Certificate Directory for Docker mc

Mount certificate directory into the container:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      -v /usr/local/share/ca-certificates/rustfs:/usr/local/share/ca-certificates/rustfs:ro \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

Note:

    Different image CA certificate paths and update methods may vary.
    --insecure can be used for experiments.
    Production should use trusted certificates and avoid long-term reliance on --insecure.

---

## SixteenI don't know.Nginx Reverse Proxy Security Parameters

### 16.1 client_max_body_size

Configuration:

    client_max_body_size 0;

Meaning:

    No limit on upload body size.

Object storage scenario:

    May upload large files.
    If the limit is too small, it will result in 413 Request Entity Too Large.

Production recommendation:

    If the business allows large objects, set to 0 or a sufficiently large value.
    If the business only allows small files, set a specific size, e.g., 200m, 1g.
    Upload size should align with business rules and security policies.

---

### 16.2 proxy_request_buffering

Configuration:

    proxy_request_buffering off;

Meaning:

    Nginx does not cache the entire request body locally before forwarding to the backend.

Recommended to disable for object storage scenarios:

    Large file uploads do not consume Nginx disk cache.
    Reduce intermediate steps for large object uploads.
    Avoid filling Nginx temporary directories.

---

### 16.3 proxy_read_timeout / proxy_send_timeout

Configuration:

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;

Purpose:

    Prevent Nginx from disconnecting during long uploads/downloads.

Production recommendation:

Set according to object size and network bandwidth.
Large object business requires longer timeouts.
Ordinary small object business can appropriately reduce.

---

### 16.4 Host Header Passthrough

Configuration:

    proxy_set_header Host $http_host;

Importance:

    S3 signatures may depend on Host.
    Presigned URLs are also related to the access domain name.
    Nginx rewriting Host may cause SignatureDoesNotMatch.

Production Recommendations:

    Keep the client's Host.
    Ensure Endpoint, certificate domain name, and Nginx server_name are consistent.
    Use externally accessible domain names when generating Presigned URLs.

---

## SeventeenI don't know.Common Security Issues Troubleshooting

### 17.1 AccessDenied

Phenomenon:

    AccessDenied
    403 Forbidden

Common Causes:

    AccessKey lacks permissions.
    SecretKey is incorrect.
    Bucket does not exist.
    Accessed a Bucket not belonging to the business.
    Current policy does not allow GetObject / PutObject / DeleteObject.
    Business account permissions are not bound or not effective.

Troubleshooting:

    mc alias list
    mc ls rustfs-admin
    mc ls rustfs-app/prod-app-uploads
    mc ls rustfs-app/prod-backups
    docker logs rustfs-cluster --tail=100

Handling:

    Confirm the used alias.
    Confirm AccessKey.
    Confirm Bucket.
    Confirm Policy.
    Validate with business account, not only with administrator account.

---

### 17.2 SignatureDoesNotMatch

Phenomenon:

    SignatureDoesNotMatch
    The request signature we calculated does not match the signature you provided

Common Causes:

    SecretKey is incorrect.
    Time is out of sync.
    Region inconsistency.
    Endpoint inconsistency.
    HTTP / HTTPS inconsistency.
    Nginx rewriting Host.
    Presigned URL generated using internal network Endpoint, accessed via external domain.
    Path-style / Virtual-hosted-style mismatch.

Troubleshooting:

    timedatectl
    curl -k -i https://s3.rustfs.local/health
    tail -100 /var/log/nginx/rustfs-s3-error.log
    docker logs rustfs-cluster --tail=200
    mc alias list

Handling:

    Correct the key.
    Synchronize time.
    Use a unified external Endpoint.
    Nginx preserves Host.
    SDK enables path-style.
    Use external HTTPS domain name when generating Presigned URLs.

---

### 17.3 Certificate Not Trusted

Phenomenon:

    certificate verify failed
    x509: certificate signed by unknown authority
    SSL certificate problem

Causes:

    Self-signed certificate is not trusted by client.
    Incomplete certificate chain.
    Access via IP, but certificate is signed for domain.
    Certificate expired.
    Domain and certificate SAN mismatch.

Troubleshooting:

    openssl s_client -connect s3.rustfs.local:443 -servername s3.rustfs.local
    curl -v https://s3.rustfs.local/health
    openssl x509 -in /etc/nginx/certs/rustfs/rustfs.local.crt -noout -dates
    openssl x509 -in /etc/nginx/certs/rustfs/rustfs.local.crt -noout -text | grep -E "DNS:|IP Address"

Handling:

    Access using correct domain.
    Import enterprise CA.
    Replace official certificate.
    Fix certificate chain.
    Renew certificate.

---

### 17.4 Nginx 413

Phenomenon:

    413 Request Entity Too Large

Causes:

    client_max_body_size is too small.

Handling:

    client_max_body_size 0;

Or:

    client_max_body_size 10g;

Reload:

    nginx -t
    systemctl reload nginx

---

### 17.5 Nginx 502

Phenomenon:

    502 Bad Gateway

Causes:

    Backend RustFS is unavailable.
    Backend port is unreachable.
    Upstream configuration error.
    Firewall interception.
    RustFS container anomaly.

Troubleshooting:

    curl -i http://10.0.0.51:9000/health
    curl -i http://10.0.0.52:9000/health
    docker ps | grep rustfs
    docker logs rustfs-cluster --tail=100
    tail -100 /var/log/nginx/rustfs-s3-error.log

---

### 17.6 Nginx 504

Phenomenon:

    504 Gateway Timeout

Causes:

    Backend response is slow.
    Large file upload/download timeout.
    proxy_read_timeout is too short.
    proxy_send_timeout is too short.
    Backend node under pressure.
    Unstable network.

Handling:

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    proxy_connect_timeout 60s;

Also check:

    Backend RustFS logs.
    Disk I/O.
    Network bandwidth.
    Nginx error.log.

---

## EighteenI don't know.Key Rotation Process

### 18.1 Why Rotate Keys

Need to rotate keys when:

Key Leakage.  
Employee Departure.  
Business Owner Change.  
Regular Security Requirements.  
Permission Convergence.  
Temporary Key Expiry.  
Audit Discovery of Abnormal Access.  

---

### 18.2 Standard Rotation Process  

Recommended Process:  

    1. Create a new AccessKey.  
    2. Bind the same or converged permissions.  
    3. Deliver the new key securely to the business.  
    4. Business performs gray-scale configuration update.  
    5. Retain the old key for a period of time.  
    6. Monitor access logs.  
    7. Confirm full business switch.  
    8. Delete the old AccessKey.  
    9. Record rotation time and responsible person.  

---

### 18.3 Prohibited Actions  

Not recommended:  

    Directly delete old keys and notify the business.  
    Send SecretKey in plain text via group chats.  
    Commit new keys to Git.  
    Share a single key across multiple businesses.  
    Delete old keys without confirming business switch completion.  

---

## NineteenI don't know.Production Security Baseline  

### 19.1 Authentication and Authorization  

| Check Item | Requirements |  
|---|---|  
| Administrator Key | Only for operations, not for business |  
| Business Key | Independent per business |  
| Environment Isolation | dev/test/prod separated |  
| Bucket Isolation | Different buckets for different businesses |  
| Minimum Permissions | Only necessary actions |  
| Delete Permissions | Requires separate approval |  
| Key Rotation | Has a process |  
| Key Storage | Secure channels or key system |  
| Privilege Verification | Must test failure scenarios |  

---

### 19.2 Network and Entry Points  

| Check Item | Requirements |  
|---|---|  
| External Access | HTTPS |  
| Backend Nodes | Not exposed to public internet |  
| API Entry | Unified domain |  
| Console Entry | Separate domain |  
| Console Sources | Restrict to operation network segments |  
| Internal HTTP | Only trusted networks |  
| Firewall | Restrict sources |  
| Nginx Logs | Enabled |  
| Upload Limits | Clearly configured |  
| Timeout | Adapted for large objects |  

---

### 19.3 Certificates and TLS  

| Check Item | Requirements |  
|---|---|  
| Certificate Source | Formal CA or enterprise CA |  
| Private Key Permissions | 600 |  
| Certificate Chain | Complete |  
| Domain Match | Mandatory |  
| Expiry Monitoring | Mandatory |  
| TLS Version | TLSv1.2+ |  
| Self-signed Certificates | Only for experiments |  
| --insecure | Only for temporary troubleshooting |  

---

### 19.4 Audit and Logs  

| Check Item | Requirements |  
|---|---|  
| Nginx access.log | Retained |  
| Nginx error.log | Retained |  
| RustFS Logs | Retained |  
| Console Login | Traceable |  
| AccessKey Creation | Recorded |  
| AccessKey Deletion | Recorded |  
| Bucket Deletion | Requires approval |  
| Large Object Deletion | Alert |  
| 403/5xx Abnormalities | Alert |  

---

## TwentyI don't know.Production Fault Record Template  

| Item | Content |  
|---|---|  
| Fault Time |  |  
| Discovery Method | Alert / User Feedback / Patrol |  
| Affected Business |  |  
| Endpoint |  |  
| Bucket |  |  
| AccessKey Name |  |  
| Error Phenomenon | AccessDenied / SignatureDoesNotMatch / 502 / 504 / Certificate Error |  
| Impact on Upload | Yes / No |  
| Impact on Download | Yes / No |  
| Nginx Logs |  |  
| RustFS Logs |  |  
| Certificate Status |  |  
| Permission Change Records |  |  
| Handling Actions |  |  
| Recovery Time |  |  
| Root Cause |  |  
| Subsequent Improvements |  |  

---

## Twenty-oneI don't know.Experiment Cleanup  

### 21.1 Delete Test Objects  

Execute:  

    docker run --rm \  
      -v /data/rustfs-security/mc-config:/root/.mc \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      --insecure rm --recursive --force rustfs-https/prod-app-uploads/rustfs-https-test.txt  

---

### 21.2 Delete Test Bucket  

High-risk warning:  

    Do not delete Buckets in production environments arbitrarily.  
    Delete only after confirming no use.  

Delete:  

    docker run --rm \  
      -v /data/rustfs-security/mc-config:/root/.mc \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      --insecure rb rustfs-https/prod-app-uploads  

If the Bucket is not empty, delete objects first.  

---

### 21.3 Delete Nginx Configuration  

Execute on rustfs-entry:  

    rm -f /etc/nginx/conf.d/rustfs-s3-https.conf  
    rm -f /etc/nginx/conf.d/rustfs-console-https.conf  

Check:  

    nginx -t  

Reload:  

    systemctl reload nginx  

---

### 21.4 Delete Experimental Self-signed Certificates  

After confirming no further use:  

    rm -rf /etc/nginx/certs/rustfs  

---

### 21.5 Delete Test Files  

Execute on rustfs-client:  

    rm -rf /tmp/rustfs-security-test  
    rm -f /tmp/app-key-upload.txt  
    rm -f /tmp/app-key-download.txt  
    rm -f /tmp/rustfs-https-test.txt  
    rm -f /tmp/rustfs-https-download.txt  

---

### 21.6 Delete mc Configuration  

If continuing with 08 Operations and Troubleshooting experiments, recommend retaining:  

    /data/rustfs-security/mc-config  

If confirmed no longer needed:  

    rm -rf /data/rustfs-security/mc-config  

---

## Twenty-twoI don't know.Completion Criteria for This Article's Hands-on Practice  

After completing this article, the following should be at least achieved:

| Project | Standard |
|---|---|
| Permission Understanding | Clearly distinguish between administrator keys and business keys |
| AccessKey | Able to create and delete via Console |
| Bucket Planning | Able to split Buckets by environment and business |
| Minimum Permissions | Able to design read-only, read-write, and delete permissions |
| HTTPS Entry | s3.rustfs.local accessible via HTTPS |
| Console Entry | console.rustfs.local accessible via HTTPS |
| Nginx Configuration | API and Console reverse proxy by subdomain |
| Certificate | Able to generate self-signed certificates and replace official certificates |
| mc Access | Able to access via HTTPS alias |
| Large File Parameters | Nginx supports large object uploads |
| Console Source | Understand production requires restricting maintenance sources |
| Security Troubleshooting | Able to troubleshoot AccessDenied, SignatureDoesNotMatch, certificate errors, 502, 413, 504 |
| Security Baseline | Able to form production inspection checklist |

---

## 23. Interview Answer Approach

If asked in an interview:

    How would you design permissions and HTTPS entry points for RustFS object storage?

You could answer:

    I would divide RustFS security into several layers: authentication authorization, key management, Bucket permissions, HTTPS entry, reverse proxy, management entry isolation, and log auditing.
    First, administrator AccessKey is only used for initialization, creating Buckets, creating business AccessKeys, and operations management, and should not be used long-term by business. Business should use independent AccessKey / SecretKey, and isolate by business, environment, and Bucket, such as prod-app-uploads, prod-backups, prod-logs-archive. Ordinary upload business should only grant GetObject, PutObject, ListBucket permissions, and DeleteObject requires separate evaluation, and DeleteBucket should not be granted to business accounts.
    Second, external access must use HTTPS. Object storage requests will carry signature information, Presigned URL, and business object content. If external access uses HTTP, there is a risk of leakage. Internal nodes can use HTTP within trusted networks, but backend 9000 and 9001 ports should not be exposed to the public.
    Third, at the entry layer, I would use Nginx or LB to terminate TLS uniformly. For example, s3.rustfs.local for S3 API, console.rustfs.local for Console management entry. Nginx forwards to RustFS nodes at 10.0.0.51 to 10.0.0.54. S3 API configuration should pay attention to client_max_body_size, proxy_request_buffering off, proxy_read_timeout, proxy_send_timeout, and Host header transmission, otherwise large object uploads or S3 signature may fail.
    Console management entry must be restricted to sources, only allowing maintenance network segments, VPN, or bastion host access, and should not be exposed to the public. Certificate-wise, experiments can use self-signed certificates, production must use trusted certificates or enterprise CA, private key permissions should be controlled, and certificate expiration should be monitored.
    Troubleshooting: AccessDenied usually check AccessKey, Bucket, and Policy; SignatureDoesNotMatch usually check SecretKey, time synchronization, Region, Endpoint, Host header, Path-style, and HTTP/HTTPS consistency; Nginx 413 check upload size limits; 502 check if backend RustFS is available; 504 check timeout, backend pressure, and network.
    Overall principle: administrator accounts do not access business, business keys follow minimum permissions, external HTTPS, Console isolation, backend ports not exposed to public, logs and auditing must be retained.

---

## 24. Summary of This Article

This article completes RustFS permission and security learning:

1. RustFS security is not just about setting administrator passwords.
2. AccessKey / SecretKey are the core of object storage access authentication.
3. Administrator keys should not be used long-term by business.
4. Business should use independent AccessKey.
5. Different environments should use different Buckets and different keys.
6. Different businesses should use different Buckets and different keys.
7. Permission design should follow the principle of minimum permissions.
8. DeleteObject permissions must be granted cautiously.
9. DeleteBucket should not be granted to ordinary business accounts.
10. External access must use HTTPS.
11. Internal HTTP is only suitable for trusted networks.
12. Backend RustFS ports should not be exposed to the public.
13. Nginx terminating TLS is a common unified entry method.
14. S3 API and Console are recommended to use different domains.
15. Console management entry must restrict sources.
16. Self-signed certificates are only suitable for experiments.
17. Production must use trusted certificates or enterprise CA.
18. Nginx needs to configure upload size, proxy buffering, and timeouts.
19. Host header transmission is important for S3 signature.
20. --insecure is only suitable for experiments and temporary troubleshooting.
21. Key rotation must have a process.
22. Security logs and access auditing must be retained.
23. The next article will learn RustFS operations and troubleshooting: logs, capacity, node anomalies and recovery.

---

## 25. Reference Documents

RustFS Official Website:

    https://rustfs.com/

RustFS Official Documentation:

    https://docs.rustfs.com/

RustFS Security Checklist:

    https://docs.rustfs.com/installation/checklists/security-checklists.html

RustFS Access Key Management:

    https://docs.rustfs.com/administration/iam/access-token.html

RustFS TLS Configuration:

    https://docs.rustfs.com/integration/tls-configured.html

RustFS Nginx Reverse Proxy Configuration:

    https://docs.rustfs.com/integration/nginx-reverse-proxy-configuration/

RustFS S3 Compatibility:

    https://docs.rustfs.com/features/s3-compatibility/

RustFS mc Client:

    https://docs.rustfs.com/developer/mc.html

AWS S3 API:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

AWS S3 Presigned URL:

    https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html

AWS S3 Identity and Access Management:

    https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-access-control.html

Nginx Official Documentation:

    https://nginx.org/en/docs/

OpenSSL Official Documentation:

    https://www.openssl.org/docs/