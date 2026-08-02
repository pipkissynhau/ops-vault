# RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxies

Recommended Path: 05-Storage/04-RustFS/07-RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxies.md

Tags: #RustFS #Object Storage #S3 #Access Keys #Permission Control #HTTPS #TLS #Nginx #Reverse Proxies #Certificates #Security Baselines #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the seventh in the RustFS series, focusing on learning about RustFS’s permissions, security, access keys, HTTPS, TLS, Nginx reverse proxies, and production security baselines.

What has been covered previously includes:

    01-RustFS Basics: S3-compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Understanding Single-Node and Cluster Modes
    03-RustFS Single-Node Deployment Practice: Service Startup, Data Directory, and Access Verification
    04-RustFS Cluster Deployment Practice: Multiple Nodes, Multiple Disks, and Access Points
    05-RustFS vs. MinIO: Architecture, Deployment, Ecosystem, and Operational Differences
    06-RustFS Client Access: S3 API, mc Tools, and Application Integration

This article addresses the following key topics:

    Why RustFS must have a secure design
    How to manage AccessKey/SecretKey pairs
    How to distinguish between administrator keys and business keys
    Understanding the concept of least privilege
    How to plan Bucket permissions
    Why external access must use HTTPS
    When to choose internal HTTP over external HTTPS
    The differences between directly enabling TLS in RustFS and terminating TLS through Nginx
    How to use Nginx to provide a unified HTTPS entry point
    How to separate the S3 API domain name from the Console management domain name
    How to configure certificates
    How to set up mc to use an HTTPS endpoint
    How to troubleshoot certificate errors, reverse proxy issues, AccessDenied, and SignatureDoesNotMatch errors
    How to establish a production security checklist for RustFS

The experimental approach adopted in this article includes:

    Continuing to use internal HTTP for backend RustFS nodes
    Providing HTTPS through a unified Nginx entry point for external access
    Using different domain names for the S3 API and Console
    Restricting the sources of management access
    Using separate AccessKeys for business operations
    Not providing administrator AccessKeys directly for business use

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the core principles of RustFS security design.
2. Comprehend the role of AccessKey/SecretKey pairs.
3. Distinguish between administrator keys and business keys.
4. Create business AccessKeys through the RustFS Console.
5. Plan permissions for different business Buckets.
6. Design a least privilege strategy for business accounts.
7. Understand why SecretKeys should not be committed to Git.
8. Recognize the necessity of HTTPS for external access.
9. Differentiate between internal HTTP and external HTTPS deployment scenarios.
10. Configure HTTPS for the RustFS S3 API using Nginx.
11. Set up HTTPS for the RustFS Console using Nginx.
12. Generate self-signed experimental certificates.
13. Replace experimental self-signed certificates with official ones.
14. Configure automatic HTTP redirection to HTTPS.
15. Adjust Nginx settings for upload size limits, timeouts, Host headers, and proxy buffering.
16. Use mc to access RustFS via HTTPS.
17. Troubleshoot certificate trust issues.
18. Diagnose Nginx errors such as 502, 413, and 504.
19. Resolve AccessDenied and SignatureDoesNotMatch issues.
20. Establish a production security checklist for RustFS.

---

## III. Key Conclusions

### 3.1 RustFS Security Does Not Involve Just Setting One Password

Object storage security includes at least the following aspects:

    Management of AccessKey/SecretKey pairs
    Protection of administrator accounts
    Isolation of business accounts
    Permission isolation for Buckets
    Principle of least privilege
    HTTPS
    Certificate management
    Reverse proxy security
    Isolation of management access points
    Upload size limits
    Timeout controls
    Access logs
    Error logs
    Key rotation
    Measures for handling key leaks
    Control over deletion permissions
    Auditing and alerts

Using just one administrator password shared by all business accounts is an extremely unsafe practice.

---

### 3.2 Administrator Keys Should Not Be Used Long-Term for Business Purposes

Administrator keys are used for:

    Initializing RustFS
    Creating AccessKeys
    Creating Buckets
    Configuring permissions
    Operational management
    Emergency handling

Business keys are used```markdown
Not for general business use.
Not to be written into application configuration.
Not to be written into common CI/CD variables.
Only to be stored through secure channels.
Regularly rotated.
Immediately disabled or replaced in case of leakage.
---

### 5.2 Business Access Keys

Business Access Keys are used for:

- Specific businesses
- Specific environments
- Specified Buckets
- Specified actions

Examples:

| Business | Environment | Bucket | Permissions |
|-----------|--------------|----------|---------------|
| app-api | dev        | dev-app-uploads | GetObject / PutObject / ListBucket |
| app-api | prod        | prod-app-uploads | GetObject / PutObject / ListBucket |
| backup-job | prod        | prod-backups | GetObject / PutObject / ListBucket |
| log-archive | prod        | prod-logs-archive | PutObject / ListBucket |
| ai-job | test       | test-ai-datasets | GetObject / PutObject / ListBucket |

It is not recommended to:

- Share one Access Key across all businesses.
- Use the same Access Key for dev, test, and prod environments.
- Give a business account permission to access all Buckets.
- Grant a business account the power to delete Buckets.
- Assign a business account administrative privileges.

---

### 5.3 Principle of Least Privilege

The principle of least privilege means:

- Only allow necessary operations.
- Only permit access to necessary Buckets.
- Use specific Prefixes for resources.
- Deny unnecessary actions by default.

Common operations include:

- s3:ListBucket
- s3:GetObject
- s3:PutObject
- s3:DeleteObject
- s3:GetBucketLocation

For regular upload tasks:

- ListBucket
- GetObject
- PutObject

Deciding whether to grant the DeleteObject permission requires careful consideration.

For backup tasks:

- PutObject
- ListBucket
- GetObject

For backup cleanup tasks:

- DeleteObject

It is advisable to manage the DeleteObject permission separately and not assign it by default to all businesses.

---

## VI. Creating Access Keys via the Console

### 6.1 Logging in to the Console

Access address:

    https://console.rustfs.local/rustfs/console

If HTTPS is not configured for testing purposes, you can use:

    http://10.0.0.51:9001/rustfs/console

Login credentials:

- AccessKey: rustfsadmin
- SecretKey: RustFSAdmin@123456

Production notes:

- Do not use test keys in production.
- The Console should not be exposed to the public internet.
- Access to the Console should be secured through VPN, bastion hosts, or dedicated network segments.
- HTTPS must be used for accessing the Console.

---

### 6.2 Creating an Access Key

Steps to create:

- Go to the Console
- Select "Access Keys"
- Click "Add Access Key"

Recommended fields to fill in:

- Name: app-api-prod
- Description: Access key for prod app uploads bucket
- Expiration: Set according to current security policies

After creation, save immediately:

- AccessKey
- SecretKey

Notes:

- The SecretKey is usually displayed only once.
- Once created, it must be stored through secure channels.
- Do not share screenshots or post the keys in public documents or repositories.

---

### 6.3 Deleting an Access Key

Steps to delete:

- Go to the Console
- Select "Access Keys"
- Choose the target Access Key
- Click "Delete"

Scenarios where deletion is appropriate:

- Key leakage
- Business decommissioning
- Employee resignation
- After key rotation
- When permissions need to be reconfigured

Before deleting, confirm:

- Whether the key is still in use by any business.
- If a new key has been assigned.
- Whether deletion will affect production activities.
- Whether all relevant parties have been notified.

---

### 6.4 Access Key Record Template

| Item          | Content                                                                 |
|---------------|----------------------------------------------------------------------------------------------------------------------------|
| AccessKey Name | app-api-prod                                                                 |
| Business        | app-api                                                                 |
| Environment     | prod                                                                 |
| Bucket         | prod-app-uploads                                                                 |
| Permission Scope | GetObject / PutObject / ListBucket                                                                 |
| Object Deletion   | No                                                                 |
| Bucket Deletion  | No                                                                 |
| Creation Time  |                                                                 |
| Expiration Date |                                                                 |
| Responsible Person |                                                                 |
| Storage Location | Key Management System                                                                 |
| Rotation Interval |                                                                 |
| Last Rotation    |                                                                 |
| Remarks        |                                                                 |
```

---

## VII. Bucket and Permission Planning

### 7.1 Bucket Naming Convention

Recommended names:

- dev-app-uploads
- test-app-uploads
- prod-app-uploads
- prod-backups
- prod-logs-archive
- prod-ai-datasets```json
{
  "Rules": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::prod-app-uploads"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": ["arn:aws:s3:::prod-app-uploads/*"]
    }
  ]
}
```

High-Risk Reminder:

The `DeleteObject` permission should be granted with caution. It is best to separate it from regular upload permissions. Deletion operations in production environments should be subject to auditing and recycling policies. It is not recommended that business accounts have default access to delete buckets.

---

### 7.7 Verification of Current Version Permission Functions

Since RustFS is still evolving rapidly, the behavior of IAM, Policies, and AccessKeys may vary across different versions.

Before going into production, it is essential to verify the following:

- Whether the current version supports the required policies.
- That the policy syntax is fully compatible with S3.
- That AccessKeys can be assigned the specified permissions.
- That read-only policies function correctly.
- That read-write policies function correctly.
- That delete permissions can be controlled independently.
- That bucket-level permissions meet expectations.
- That prefix-level permissions meet expectations.
- That the behavior of console and API permissions is consistent.

Verification Guidelines:

- Do not test only using an administrator account; always use a business account for verification.
- Ensure that attempts to access with unauthorized privileges fail.
- Verify that delete permissions are properly restricted.
---

## Section 8: Practical Exercise 1: Creating a Business Bucket

### 8.1 Configuring the mc Administrator Alias

Execute the following in `rustfs-client`:

```bash
mkdir -p /data/rustfs-security/mc-config
```

Configure the administrator alias:

```bash
docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  alias set rustfs-admin http://s3.rustfs.local:9000 rustfsadmin 'RustFSAdmin@123456'
```

Verify the configuration:

```bash
docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-admin
```

---

### 8.2 Creating a Production Simulation Bucket

Create the bucket:

```bash
docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mb rustfs-admin/prod-app-uploads
```

Create a backup bucket:

```bash
docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mb rustfs-admin/prod-backups
```

Create a log archiving bucket:

```bash
docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mb rustfs-admin/prod-logs-archive
```

Verify the buckets:

```bash
docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-admin
```

---

### 8.3 Uploading Test Objects

Prepare the files:

```bash
mkdir -p /tmp/rustfs-security-test
cd /tmp/rustfs-security-test
echo "security test object" > hello.txt
echo "backup data" > backup.sql
echo "nginx access log" > access.log
```

Upload the files:

```bash
docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  -      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 9.3 Verify that the business account can access the target Bucket

Execute:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-app/prod-app-uploads

Upload a test file:

    echo "upload by app key" > /tmp/app-key-upload.txt

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /tmpdata/app-key-upload.txt rustfs-app/prod-app-uploads/app-key-upload.txt

Download a test file:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp rustfs-app/prod-app-uploads/app-key-upload.txt /tmpdata/app-key-download.txt

Verify:

    cat /tmp/app-key-download.txt

---

### 9.4 Verify that the business account should not access other Buckets

Try to access the backup Bucket:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-app/prod-backups

Expected result:

    If the permission settings are correct, it should return AccessDenied or an error indicating no permissions.

Try to access the log archive Bucket:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-app/prod-logs-archive

Expected result:

    The business account should not have access to unrelated Buckets.

---

### 9.5 Verify deletion permissions

If the business account is not authorized to delete objects, execute:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm rustfs-app/prod-app-uploads/app-key-upload.txt

Expected result:

    If the DeleteObject permission is not granted, it should return AccessDenied.

If the business requires deletion permissions, the following aspects should be confirmed:

    Whether object deletion is audited.
    Whether there is a recycle bin or version control system in place.
    Whether backups are available.
    Whether the deletion has been approved by the responsible person.
    Whether batch deletions are allowed.

---

## Section X: HTTPS Implementation Solutions

### 10.1 Solution 1: RustFS enables TLS directly

RustFS supports providing HTTPS through TLS-related configurations.

Typical steps:

    Prepare rustfs_cert.pem and rustfs_key.pem.
    Mount the certificate directory.
    Set the RUSTFS_TLS_PATH variable.
    Restart RustFS.

Advantages:

    The RustFS process itself provides HTTPS.
    Backend transmissions can also be encrypted.
    A single layer of reverse proxy is eliminated.

Disadvantages:

    Managing certificates across multiple nodes is complex.
    Certificate updates require handling on every node.
    Console and API access control is not as flexible as with Nginx.
    Additional design is needed for upload size limitations, timeouts, access logging, and origin filtering.
    If there is an LB in front, unified access control is still required.

Suitable for:

    Situations where full-chain TLS is required internally.
    Organizations with the capability to manage certificates centrally.
    Those with advanced configuration management capabilities.
    Applications that need to communicate over untrusted networks.

---

### 10.2 Solution 2: Nginx terminates TLS

It is recommended to start with this solution in experimental and small-to-medium-sized environments:

/etc/nginx/certs/rustfs/privkey.pem            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            proxy_pass http://rustfs_console_backend;
        }
    }
    EOF

---

### 13.3 Checking and Reloading Nginx

Execute:

    nginx -t
    systemctl reload nginx

Check the logs:

    tail -50 /var/log/nginx/rustfs-console-error.log

---

### 13.4 Testing Console Access

Access via browser:

    https://console.rustfs.local/rustfs/console

If it's a self-signed certificate:

    The browser will indicate that the certificate is not trusted.
    In a testing environment, you can proceed manually.
    In a production environment, you must use a trusted certificate or an enterprise CA.

Login:

    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456

---

### 13.5 Restricting Console Access by IP Source

It is recommended in production to only allow access from the operations network segment.

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

        proxy_pass http://rustfs-console_backend;
    }

Explanation:

    10.0.0.0/24 represents the current testing network segment.
    In production, this should be replaced with the IP address of the bastion host, VPN, or operations exit point.
    It is not recommended to expose the Console to all public sources.

---

## Chapter Fourteen: Practical Exercise Six: Configuring mc Access to RustFS via HTTPS

### 14.1 Accessing with a Self-Signed Certificate

Since self-signed certificates are used in this experiment, mc may not trust the certificate.

You can start by using --insecure for testing.

Configure an alias:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set rustfs-https https://s3.rustfs.local rustfsadmin 'RustFSAdmin@123456'

Verify:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-https

---

### 14.2 Uploading Objects via HTTPS

Create a file:

    echo "upload through rustfs https entry" > /tmp/rustfs-https-test.txt

Upload:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /tmpdata/rustfs-https-test.txt rustfs-https/prod-app-uploads/rustfs-https-test.txt

Verify:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-https/prod-app-uploads

---

### 14.3 Downloading and Verifying

Download:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp rustfs-https/prod-app-uploads/rustfs-https-test.txt /tmpdata/rustfs-https-download.txt

Verify:

    cat /tmp/rustfs-https-download.txt

    sha256sum /tmp/rustfs-https-test.txt /tmp/rustfs-https-download.txt

If the business only allows small files, a specific size can be set, such as 200MB or 1GB. The upload size should be determined in conjunction with business rules and security policies.

---

### 16.2 proxy_request_buffering

Configuration:

    proxy_request_buffering off;

Meaning:

    Nginx does not cache the entire request body locally before forwarding it to the backend.

It is recommended to turn this off in object storage scenarios because:

- Large file uploads do not utilize Nginx's disk cache.
- It reduces intermediate steps in large object uploads.
- It prevents Nginx's temporary directories from becoming full.

---

### 16.3 proxy_read_timeout / proxy_send_timeout

Configuration:

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;

Function:

    This setting prevents Nginx from prematurely disconnecting during large file uploads or downloads.

Production recommendations:

- Adjust these values based on the size of objects and network bandwidth.
- Large object services require longer timeouts.
- For regular small object services, these times can be reduced appropriately.

---

### 16.4 Host head透传

Configuration:

    proxy_set_header Host $http_host;

Importance:

- S3 signatures may depend on the Host header.
- Presigned URLs are also related to the domain name used for access.
- If Nginx modifies the Host header, it may cause a SignatureDoesNotMatch error.

Production recommendations:

- Retain the client's original Host header.
- Ensure that the Endpoint, certificate domain name, and Nginx server_name are consistent.
- Use an externally accessible domain name when generating Presigned URLs.

---

## Chapter Seventeen: Common Security Issues and Troubleshooting

### 17.1 AccessDenied

Symptoms:

    AccessDenied
    403 Forbidden

Common causes:

- The AccessKey does not have the necessary permissions.
- The SecretKey is incorrect.
- The specified Bucket does not exist.
- An attempt is being made to access a Bucket that is not part of this business.
- Current policies do not allow operations such as GetObject, PutObject, or DeleteObject.
- The business account's permissions are not properly assigned or have not taken effect.

Troubleshooting steps:

    - Check the list of aliases used.
    - Verify the AccessKey.
   - Confirm the existence of the Bucket.
   - Review the current policies.
   - Verify using a business account, not just an administrative one.

---

### 17.2 SignatureDoesNotMatch

Symptoms:

    SignatureDoesNotMatch
    The calculated signature does not match the one provided by the user.

Common causes:

- The SecretKey is incorrect.
- There is a time discrepancy between systems.
- The Regions used are different.
- The Endpoints are inconsistent.
- The HTTP or HTTPS protocols are different.
- Nginx has modified the Host header.
- A Presigned URL was generated using an internal Endpoint, but it is being accessed from an external domain name.
- The path-style or virtual-hosted-style configurations do not match.

Troubleshooting steps:

    - Check the current time setting using timedatectl.
    - Verify the status of S3 by accessing https://s3.rustfs.local/health.
   - Review the Nginx error logs at /var/log/nginx/rustfs-s3-error.log.
   - Check the Docker logs for related errors at rustfs-cluster --tail=200.
   - Verify the list of aliases used.

---

### 17.3 Certificate Not Trusted

Symptoms:

    certificate verify failed
    x509: certificate signed by unknown authority
    SSL certificate problem

Causes:

- The self-signed certificate is not trusted by clients.
- The certificate chain is incomplete.
- An IP address is being used for access, but the certificate is issued for a domain name.
- The certificate has expired.
- The domain name and the SAN fields in the certificate do not match.

Troubleshooting steps:

- Verify the connection using openssl s_client -connect s3.rustfs.local:443 -servername s3.rustfs.local.
- Check the status of S3 by accessing https://s3.rustfs.local/health.
- Use openssl x509 to verify the certificate details, including the certificate chain and domain name matching.
- If necessary, import a corporate CA certificate or replace the current self-signed certificate.
- Ensure that the certificate chain is complete and valid.

---

### 17.4 Nginx 413

Symptom:

    413 Request Entity Too Large

Cause:

- The value of client_max_body_size is set too low.

Solution:

- Increase the value of client_max_body_size, for example, to 10GB.
- To apply the changes, reload Nginx using nginx -t followed by systemctl reload nginx.

---

### 17| Nginx access.log | Retained |
| Nginx error.log | Retained |
| RustFS logs | Retained |
| Console log-in | Tracable |
| AccessKey creation | Recorded |
| AccessKey deletion | Recorded |
| Bucket deletion | Approved |
| Large-scale object deletion | Alarmed |
| 403/5xx errors | Alarmed |

---

## Twenty, Production Fault Recording Template

| Item | Details |
|---|---|
| Fault Time | |
| Discovery Method | Alarm / User Report / Inspection |
| Business Impact | |
| Endpoint | |
| Bucket | |
| AccessKey Name | |
| Error Description | AccessDenied / SignatureDoesNotMatch / 502 / 504 / Certificate Error |
| Affects Upload | Yes / No |
| Affects Download | Yes / No |
| Nginx Logs | |
| RustFS Logs | |
| Certificate Status | |
| Permission Change Record | |
| Action Taken | |
| Recovery Time | |
| Root Cause | |
| Future Improvements | |

---

## Twenty-One, Experiment Cleanup

### 21.1 Delete Test Objects

Execute:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force rustfs-https/prod-app-uploads/rustfs-https-test.txt

---

### 21.2 Delete Test Bucket

High-risk Warning:

    Do not delete Buckets in a production environment without caution.
    Only delete them after confirming they are no longer needed for testing.

Execute:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb rustfs-https/prod-app-uploads

If the Bucket is not empty, delete its objects first.

---

### 21.3 Delete Nginx Configuration

Execute in rustfs-entry:

    rm -f /etc/nginx/conf.d/rustfs-s3-https.conf
    rm -f /etc/nginx/conf.d/rustfs-console-https.conf

Check:

    nginx -t

Reload configuration:

    systemctl reload nginx

---

### 21.4 Delete Experimental Self-Signed Certificates

After confirming they are no longer needed:

    rm -rf /etc/nginx/certs/rustfs

---

### 21.5 Delete Test Files

Execute in rustfs-client:

    rm -rf /tmp/rustfs-security-test
    rm -f /tmp/app-key-upload.txt
    rm -f /tmp/app-key-download.txt
    rm -f /tmp/rustfs-https-test.txt
    rm -f /tmp/rustfs-https-download.txt

---

### 21.6 Delete mc Configuration

If you plan to continue with 08 Ops and troubleshooting experiments in the future, it is recommended to keep:

    /data/rustfs-security/mc-config

Otherwise, delete it completely:

    rm -rf /data/rustfs-security/mc-config

---

## Twenty-Two, Completion Standards for This Article

After completing this article, you should have achieved at least the following:

| Item | Standard |
|---|---|
| Permission Understanding | Clearly distinguish between administrator keys and business keys |
| AccessKey Management | Be able to create and delete AccessKeys through the Console |
| Bucket Planning | Know how to divide Buckets based on environment and business needs |
| Minimum Permissions | Understand how to design read-only, read-write, and deletion permissions |
| HTTPS Access | Ensure that s3.rustfs.local is accessible via HTTPS |
| Console Access | Ensure that console.rustfs.local is accessible via HTTPS |
| Nginx Configuration | Set up API and Console with domain-based reverse proxying |
| Certificates | Know how to generate self-signed certificates and replace them with official ones |
| mc Access | Be able to access using HTTPS aliases |
| Large File Handling | Ensure that Nginx supports large object uploads |
| Console Security | Understand the need to restrict console access from operational networks |
| Troubleshooting | Be able to resolve issues such as AccessDenied, SignatureDoesNotMatch, certificate errors, 502, 413, and 504 |
| Security Baseline | Develop a production security checklist |

---

## Twenty-Three, Interview Answer Guidelines

If you are interviewed about how to design permissions and HTTPS access for RustFS object storage, you can answer as follows:

    I would approach RustFS security from several key aspects: authentication and authorizationhttps://docs.rustfs.com/administration/iam/access-token.html

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