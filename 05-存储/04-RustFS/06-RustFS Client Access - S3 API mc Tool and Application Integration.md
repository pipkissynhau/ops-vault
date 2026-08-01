# RustFS Client Access: S3 API, mc Tools, and Application Integration

Recommended path: 05-Storage/04-RustFS/06-RustFS Client Access: S3 API, mc Tools, and Application Integration.md

Tags: #RustFS #S3 #ObjectStorage #mc #AWSCLI #SDK #PresignedURL #PathStyle #VirtualHostedStyle #ApplyAccess #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the sixth article of the RustFS module, focusing on learning RustFS client access methods.

Previously completed:

    01-RustFS Basics: S3-compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Single-node and Cluster Modes
    03-RustFS Single-node Deployment Practice: Service Startup, Data Directory, and Access Verification
    04-RustFS Cluster Deployment Practice: Multi-node, Multi-disk, and Access Entry
    05-RustFS vs MinIO: Architecture, Deployment, Ecosystem, and Operations Differences

This article focuses on solving:

    How RustFS clients access
    What is an S3 Endpoint
    How to configure AccessKey / SecretKey
    How mc connects to RustFS
    How mc creates a Bucket
    How mc uploads/downloads Objects
    How AWS CLI accesses RustFS
    How applications integrate with RustFS
    What is the difference between Path-style and Virtual-hosted-style
    What is a Presigned URL
    How to generate temporary download links
    What to note for large object uploads
    Why Multipart Upload must be validated
    How to troubleshoot SignatureDoesNotMatch
    How to troubleshoot AccessDenied
    How to troubleshoot Endpoint errors
    How to provide a RustFS integration parameter template for business

This article assumes RustFS has already been deployed.

You can use a single-node RustFS:

    http://10.0.0.51:9000

Or use a cluster unified entry point:

    http://s3.rustfs.local:9000

Production recommends using:

    https://s3.rustfs.local

HTTPS certificates, reverse proxies, and permission security will be further expanded in section 07.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand RustFS's S3 client access model.
2. Understand the roles of Endpoint, Bucket, Object Key, AccessKey, SecretKey, and Region.
3. Configure RustFS alias using mc.
4. Create a Bucket using mc.
5. Upload objects using mc.
6. Download objects using mc.
7. View object metadata using mc.
8. Delete objects and Buckets using mc.
9. Access RustFS using AWS CLI.
10. Upload/download objects using AWS CLI.
11. Generate Presigned URLs.
12. Validate Presigned URLs using curl.
13. Understand Path-style vs Virtual-hosted-style.
14. Configure S3 Endpoint for applications.
15. Specify required parameters for application integration with RustFS.
16. Troubleshoot SignatureDoesNotMatch.
17. Troubleshoot AccessDenied.
18. Troubleshoot EndpointConnectionError.
19. Understand differences between mc, AWS CLI, and SDK client validations.
20. Form a client compatibility verification checklist before production integration.

---

## III. RustFS Client Access Model

### 3.1 Parameters Required for Client Access

Client access to RustFS typically requires the following parameters:

| Parameter | Description | Example |
|---|---|---|
| Endpoint | RustFS S3 API address | http://10.0.0.51:9000 |
| AccessKey | Access account | rustfsadmin |
| SecretKey | Access secret key | RustFSAdmin@123456 |
| Bucket | Object storage space | app-uploads |
| Object Key | Object name | images/avatar-001.png |
| Region | Region parameter | us-east-1 |
| Path-style | Access path style | http://endpoint/bucket/key |
| HTTPS | Whether to use TLS | Must be used in production |

Experimental parameter example:

    Endpoint: http://10.0.0.51:9000
    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456
    Region: us-east-1
    Bucket: app-uploads

Cluster unified entry point example:

    Endpoint: http://s3.rustfs.local:9000
    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456
    Region: us-east-1
    Bucket: app-uploads

Production HTTPS example:

    Endpoint: https://s3.rustfs.local
    AccessKey: <Business-specific AccessKey>
    SecretKey: <Business-specific SecretKey>
    Region: us-east-1
    Bucket: app-uploads

---

### 3.2 Understanding Endpoint

Endpoint is the entry point for object storage services.

Single-node RustFS:

    http://10.0.0.51:9000

RustFS cluster any node:

    http://10.0.0.51:9000
    http://10.0.0.52:9000
    http://10.0.0.53:9000
    http://10.0.0.54:9000

RustFS unified entry point:

    http://s3.rustfs.local:9000

Production recommendation:

    https://s3.rustfs.local

Production recommendations:

    Applications should only configure the unified entry point.
    Business should not directly perceive backend nodes.
    Backend node changes should not affect application configuration.
    External access must use HTTPS.
    Backend node ports should not be exposed directly to the public.

---

### 3.3 Understanding Bucket

Bucket is the top-level space for object storage.

Common Buckets:

    app-uploads
    backups
    logs-archive
    devops-artifacts
    ai-datasets
    model-files

Recommendations:

    Each business should have at least one independent Bucket.
    Different environments should use separate Buckets.
    Different permission boundaries should use separate Buckets.
    Bucket names should be lowercase, numeric, and hyphenated.
    Avoid using Chinese, spaces, or underscores.

### 3.4 Understanding Object Keys

Object Key is the name of an object.

Examples:

    images/avatar-001.png
    resumes/user-1001/resume.pdf
    backups/mysql/full-2026-04-28.sql.gz
    logs/nginx/2026/04/28/access.log.gz
    models/llm/model-v1.bin

Notes:

    Object storage has no traditional directories.
    logs/nginx/2026/04/28/access.log.gz is a complete Object Key.
    Clients display / as directory-like effects.
    Prefix is the prefix of an Object Key, not a traditional file system directory.

---

## Four. Experimental Environment

### 4.1 RustFS Single-Node Environment

Nodes:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Single-Node |

Endpoint:

    http://10.0.0.51:9000

Console:

    http://10.0.0.51:9001/rustfs/console

---

### 4.2 RustFS Cluster Environment

Nodes:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Node 1 |
| 10.0.0.52 | rustfs-node02 | RustFS Node 2 |
| 10.0.0.53 | rustfs-node03 | RustFS Node 3 |
| 10.0.0.54 | rustfs-node04 | RustFS Node 4 |
| 10.0.0.55 | rustfs-client | Client Test Node |
| 10.0.0.56 | rustfs-entry | Nginx Unified Entry |

Unified Entry:

    http://s3.rustfs.local:9000

Production Target:

    https://s3.rustfs.local

---

### 4.3 Client Tools

This document uses three types of clients:

    mc
    AWS CLI
    Application SDK

Preferred for experiments:

    mc

Reasons:

    mc commands are intuitive.
    Suitable for Bucket / Object management.
    Suitable for migration and mirror.
    Suitable for checking if objects exist.
    Good support for S3-compatible object storage.

AWS CLI is used for:

    Verifying standard S3 command compatibility.
    Verifying access methods closer to AWS S3 on the application side.
    Generating Presigned URLs.

SDK is used for:

    Verifying real business access capabilities.
    Cannot rely solely on mc to determine production readiness.

---

## Five. Pre-Access Checks for Clients

### 5.1 Check RustFS Service

Execute on RustFS nodes:

    docker ps | grep rustfs

Check logs:

    docker logs rustfs-single --tail=100

Or cluster mode:

    docker logs rustfs-cluster --tail=100

---

### 5.2 Check API Health Status

Single-node:

    curl -i http://10.0.0.51:9000/health

Cluster unified entry:

    curl -i http://s3.rustfs.local:9000/health

Expected:

    HTTP/1.1 200 OK

---

### 5.3 Check Network from Client to RustFS

Execute on rustfs-client:

    ping -c 3 10.0.0.51
    ping -c 3 s3.rustfs.local

Check ports:

    nc -vz 10.0.0.51 9000
    nc -vz s3.rustfs.local 9000

If no nc is available:

    apt update
    apt install -y netcat-openbsd

---

### 5.4 Check Time Synchronization

Execute on client and RustFS nodes:

    timedatectl

Focus on:

    System clock synchronized: yes

Notes:

    S3 signing depends on time.
    Large time deviations may cause SignatureDoesNotMatch.
    Clients, servers, and reverse proxy nodes should all maintain time synchronization.

---

## Six. Basic mc Client Usage

### 6.1 Prepare mc Configuration Directory

Execute on client node:

    mkdir -p /data/rustfs-client/mc-config

Test mc image:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

Notes:

    This document continues to use the mc image already synchronized in the MinIO module.
    mc can access MinIO and also RustFS and other S3-compatible object storages.

---

### 6.2 Configure Single-Node RustFS Alias

If accessing single-node RustFS:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-single http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@123456'

Check:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 6.3 Configure Cluster Unified Entry Alias

If accessing cluster unified entry:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  alias set rustfs-entry http://s3.rustfs.local:9000 rustfsadmin 'RustFSAdmin@123456'

View:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  alias list

---

### 6.4 View Bucket List

Single machine:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-single

Unified entry:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-entry

If it returns normally, it indicates:

Endpoint is correct.
AccessKey / SecretKey is correct.
RustFS API is accessible.
mc basic authentication is successful.

---

## Seven, mc Manage Bucket

### 7.1 Create Bucket

Create Bucket:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mb rustfs-entry/app-uploads

View:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-entry

---

### 7.2 Create Multiple Buckets

Execute:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mb rustfs-entry/backups

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mb rustfs-entry/logs-archive

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mb rustfs-entry/devops-artifacts

View:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-entry

---

### 7.3 Bucket Naming Guidelines

Recommended:

app-uploads
prod-app-uploads
dev-app-uploads
logs-archive
mysql-backups
gitlab-backups
ai-datasets

Not recommended:

AppUploads
app_uploads
Apply Upload
app uploads
bucket#01
test..bucket

Principles:

Lowercase.
Numbers.
Hyphens.
Has business meaning.
Do not mix all business in one Bucket.

---

### 7.4 Delete Bucket

Delete empty Bucket:

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  rb rustfs-entry/devops-artifacts

High-risk warning:

Deleting a Bucket is a high-risk operation.
Production must be approved.
Confirm the Bucket is empty or data has been backed up before deletion.
Confirm no business is using it before deletion.

---

## Eight, mc Upload and Download Object

### 8.1 Prepare Test Files

Execute on the client node:

mkdir -p /tmp/rustfs-client-demo
cd /tmp/rustfs-client-demo

Create text file:

echo "hello rustfs client access" > hello.txt

Create config file: /tmp/rustfs-client-demo/config.json

cat > app.conf <<'EOF'
    app_name=rustfs-client-demo
    env=lab
    endpoint=http://s3.rustfs.local:9000
    bucket=app-uploads
    EOF

Create log directory:

    mkdir -p logs
    echo "2026-04-28 INFO rustfs client upload test" > logs/app.log

Create JSON file:

    mkdir -p data
    cat > data/user.json <<'EOF'
    {"id":1001,"name":"ops-user","storage":"rustfs"}
    EOF

Create large file:

    dd if=/dev/zero of=file-100m.bin bs=1M count=100

Calculate checksum:

    sha256sum hello.txt app.conf logs/app.log data/user.json file-100m.bin > sha256-before.txt
    cat sha256-before.txt

---

### 8.2 Upload a Single File

Upload hello.txt:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-client-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/hello.txt rustfs-entry/app-uploads/hello.txt

Check:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-entry/app-uploads

---

### 8.3 Upload a Directory

Upload logs directory:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-client-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /demo/logs rustfs-entry/app-uploads/

Check:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-entry/app-uploads

---

### 8.4 Upload Multiple Files

Upload app.conf and user.json:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-client-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/app.conf rustfs-entry/app-uploads/config/app.conf

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-client-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/data/user.json rustfs-entry/app-uploads/data/user.json

Check:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-entry/app-uploads

---

### 8.5 Upload a Large File

Upload 100Mi file:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-client-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/file-100m.bin rustfs-entry/app-uploads/file-100m.bin

Check object information:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      stat rustfs-entry/app-uploads/file-100m.bin

---

### 8.6 Download an Object

Create download directory:

    mkdir -p /tmp/rustfs-client-download

Download hello.txt: /think

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  -v /tmp/rustfs-client-download:/download \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp rustfs-entry/app-uploads/hello.txt /download/hello.txt

Download Large Files:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    -v /tmp/rustfs-client-download:/download \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    cp rustfs-entry/app-uploads/file-100m.bin /download/file-100m.bin

---

### 8.7 Verify Downloaded Files

Run:

  sha256sum /tmp/rustfs-client-demo/hello.txt /tmp/rustfs-client-download/hello.txt
  sha256sum /tmp/rustfs-client-demo/file-100m.bin /tmp/rustfs-client-download/file-100m.bin

If the hashes match, it indicates the uploaded/downloaded content is consistent.

---

## Nine. Common mc Object Management Commands

### 9.1 List Objects

List objects in a Bucket:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    ls rustfs-entry/app-uploads

Recursive listing:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    ls --recursive rustfs-entry/app-uploads

---

### 9.2 View Object Details

View object metadata:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    stat rustfs-entry/app-uploads/hello.txt

View large files:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    stat rustfs-entry/app-uploads/file-100m.bin

---

### 9.3 Delete Single Object

Delete object:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    rm rustfs-entry/app-uploads/hello.txt

Verify deletion:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    ls rustfs-entry/app-uploads

High-risk warning:

  Deleting objects is an irreversible operation.
  Confirm object backups exist before production deletion.
  Delete permissions should not be granted to regular business accounts.

---

### 9.4 Delete Directory Prefix

Delete logs prefix:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    rm --recursive --force rustfs-entry/app-uploads/logs

High-risk warning:

  "Directories" in object storage are essentially key prefixes.
  Deleting a prefix may removeMass objects.
  Production operations must be cautious.

---

### 9.5 Mirror Object Sync

Sync local directory to RustFS:

  docker run --rm \
    -v /data/rustfs-client/mc-config:/root/.mc \
    -v /tmp/rustfs-client-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    mirror /demo rustfs-entry/app-uploads/demo-mirror

Sync from RustFS to local:

  mkdir -p /tmp/rustfs-mirror-download

docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  -v /tmp/rustfs-mirror-download:/download \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mirror rustfs-entry/app-uploads/demo-mirror /download

**Note:**

  mirror is commonly used for migration and synchronization.
  Understand the behavior of overwriting, deletion, and incremental synchronization before production use.
  Do not execute unfamiliar mirror commands directly on production Buckets.

---

## TenI don't know.AWS CLI Access to RustFS

### 10.1 Install AWS CLI

Ubuntu:

  apt update
  apt install -y awscli

Check version:

  aws --version

**Note:**

  The awscli version in the apt repository may be outdated.
  Basic s3 ls / cp / presign commands are generally sufficient for experimentation.
  Production SDK compatibility verification should use the actual business version.

---

### 10.2 Configure Environment Variables

Execute:

  export AWS_ACCESS_KEY_ID="rustfsadmin"
  export AWS_SECRET_ACCESS_KEY="RustFSAdmin@123456"
  export AWS_DEFAULT_REGION="us-east-1"

Check:

  echo ${AWS_ACCESS_KEY_ID}
  echo ${AWS_DEFAULT_REGION}

**Security Reminder:**

  Do not print SecretKey to public logs.
  Do not write SecretKey to public scripts.
  Production should use more secure key management methods.

---

### 10.3 List Buckets

Single machine:

  aws --endpoint-url http://10.0.0.51:9000 s3 ls

Unified entry point:

  aws --endpoint-url http://s3.rustfs.local:9000 s3 ls

---

### 10.4 Create Bucket

Create:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 mb s3://awscli-demo

Check:

  aws --endpoint-url http://s3.rustfs.local:9000 s3 ls

---

### 10.5 Upload Objects

Create file:

  echo "hello from aws cli" > /tmp/awscli-hello.txt

Upload:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 cp /tmp/awscli-hello.txt s3://awscli-demo/awscli-hello.txt

Check:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 ls s3://awscli-demo/

---

### 10.6 Download Objects

Download:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 cp s3://awscli-demo/awscli-hello.txt /tmp/awscli-hello-download.txt

Check:

  cat /tmp/awscli-hello-download.txt

Verify:

  sha256sum /tmp/awscli-hello.txt /tmp/awscli-hello-download.txt

---

### 10.7 Recursive Upload Directory

Create directory:

  mkdir -p /tmp/awscli-dir/logs
  echo "log from aws cli" > /tmp/awscli-dir/logs/app.log
  echo "config from aws cli" > /tmp/awscli-dir/app.conf

Upload:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 cp /tmp/awscli-dir s3://awscli-demo/awscli-dir --recursive

Check:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 ls s3://awscli-demo/ --recursive

---

### 10.8 Delete Objects

Delete single object:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 rm s3://awscli-demo/awscli-hello.txt

Recursive delete prefix:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 rm s3://awscli-demo/awscli-dir --recursive

**High-risk Reminder:**

  aws s3 rm --recursive is a high-risk command.
  Production must exercise caution.
  Confirm Bucket, Prefix, and environment before execution.

---

## ElevenI don't know.Presigned URL Verification

### 11.1 What is Presigned URL

Presigned URL is a pre-signed URL.

**Functions:**

  Provide users with a temporary access URL to objects without exposing AccessKey / SecretKey.
  The URL is valid for a specified duration.
  Commonly used for temporary downloads, temporary uploads, file sharing, and private object access authorization.

**Typical Scenarios:**

  User downloads a private attachment.
  User temporarily uploads a file.
  Backend generates URL, frontend directly accesses it.
  Reduce the pressure on application servers to forward large files.

**Notes:**

  Presigned URL itself contains signature parameters.
  URL leakage may allow access within the validity period.
  The validity period should not be too long.
  Production should combine permissions, expiration time, download range, and log auditing.

---

### 11.2 Prepare Object

Upload a test object:

  echo "presigned url test" > /tmp/presign-test.txt

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 cp /tmp/presign-test.txt s3://awscli-demo/presign-test.txt

Check:

  aws --endpoint-url http://s3.rustfs.local:9000 \
    s3 ls s3://awscli-demo/presign-test.txt

### 11.3 Generating a Download Presigned URL

Generate a 10-minute valid URL:

    aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 presign s3://awscli-demo/presign-test.txt \
      --expires-in 600

Save the output:

    PRESIGNED_URL=$(aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 presign s3://awscli-demo/presign-test.txt \
      --expires-in 600)

View the result:

    echo "${PRESIGNED_URL}"

---

### 11.4 Downloading with curl

Execute:

    curl -L "${PRESIGNED_URL}" -o /tmp/presign-test-download.txt

View the result:

    cat /tmp/presign-test-download.txt

Verify:

    sha256sum /tmp/presign-test.txt /tmp/presign-test-download.txt

---

### 11.5 Common Issues with Presigned URLs

If access fails, check:

    Whether the URL has expired.
    Whether the client's time is accurate.
    Whether the RustFS node's time is accurate.
    Whether the endpoint is accessible.
    Whether the client can resolve the domain in the URL.
    Whether HTTP or HTTPS is being used.
    Whether the reverse proxy is rewriting the Host.
    Whether the bucket and object key are correct.
    Whether the credentials used to generate the URL have GetObject permissions.

Production notes:

    If an application generates Presigned URLs externally, the endpoint must be an externally accessible domain.
    Do not generate http://10.0.0.51:9000 such internal network addresses for public users.
    Production should generate https://s3.example.com/bucket/key such accessible addresses.

---

## Twelve, Path-style vs Virtual-hosted-style

### 12.1 Path-style Access

Path-style format:

    http://s3.rustfs.local:9000/app-uploads/hello.txt

Structure:

    Endpoint: http://s3.rustfs.local:9000
    Bucket: app-uploads
    Object: hello.txt

Full path:

    /app-uploads/hello.txt

Advantages:

    Simpler to experiment with.
    No need to configure DNS for each bucket.
    Easier to verify private object storage compatibility.
    Nginx configuration is relatively simple.

---

### 12.2 Virtual-hosted-style Access

Virtual-hosted-style format:

    http://app-uploads.s3.rustfs.local:9000/hello.txt

Structure:

    Bucket: app-uploads
    Endpoint: s3.rustfs.local
    Object: hello.txt

Full domain name:

    app-uploads.s3.rustfs.local

Advantages:

    Closer to AWS S3 access style.
    Suitable for some SDK default behaviors.
    Suitable for production scenarios with more granular domain hierarchies.

Challenges:

    Requires wildcard DNS.
    Requires wildcard certificates.
    Nginx must correctly pass through the Host.
    Private environment configuration complexity is higher.

---

### 12.3 Experimental Recommendations

During the RustFS experimental phase, it is recommended to prioritize using:

    Path-style

Reasons:

    Simpler.
    More controllable.
    More suitable for internal network experiments.
    More suitable for mc / AWS CLI basic verification.
    Does not require wildcard domains or certificates.

In production, decide based on the business SDK:

    If the SDK supports forcing Path-style, continue using Path-style.
    If the SDK defaults to Virtual-hosted-style, configure DNS, certificates, and reverse proxy.
    Must validate the actual access style of the SDK before business access.

---

### 12.4 Path-style Parameters in SDKs

Common configuration approach for Python boto3:

    s3 = boto3.client(
        "s3",
        endpoint_url="https://s3.rustfs.local",
        aws_access_key_id="<ACCESS_KEY>",
        aws_secret_access_key="<SECRET_KEY>",
        region_name="us-east-1",
        config=Config(s3={"addressing_style": "path"})
    )

Common configuration approach for JavaScript AWS SDK v3:

    forcePathStyle: true

Common configuration approach for Java AWS SDK:

    pathStyleAccessEnabled: true

Notes:

    Specific code will depend on the application language and SDK version.
    Operations teams must at least know whether the application uses Path-style.
    If the application encounters access errors, Path-style is one of the key troubleshooting areas.

---

## Thirteen, Application Access Parameter Template

### 13.1 Backend Application Configuration Template

Business applications accessing RustFS typically require the following configuration:

    OBJECT_STORAGE_TYPE=s3
    OBJECT_STORAGE_ENDPOINT=https://s3.rustfs.local
    OBJECT_STORAGE_REGION=us-east-1
    OBJECT_STORAGE_BUCKET=app-uploads
    OBJECT_STORAGE_ACCESS_KEY=<Business Access Key>
    OBJECT_STORAGE_SECRET_KEY=<Business Secret Key>
    OBJECT_STORAGE_FORCE_PATH_STYLE=true
    OBJECT_STORAGE_USE_SSL=true

Experimental environment: /think

OBJECT_STORAGE_TYPE=s3
OBJECT_STORAGE_ENDPOINT=http://s3.rustfs.local:9000
OBJECT_STORAGE_REGION=us-east-1
OBJECT_STORAGE_BUCKET=app-uploads
OBJECT_STORAGE_ACCESS_KEY=rustfsadmin
OBJECT_STORAGE_SECRET_KEY=RustFSAdmin@123456
OBJECT_STORAGE_FORCE_PATH_STYLE=true
OBJECT_STORAGE_USE_SSL=false

Production Reminder:

    SecretKey should not be written directly to Git.
    Should be stored in a Secret management system.
    Kubernetes should use Secret.
    VM should use environment variable files and restrict permissions.
    Different businesses should use different AccessKey.

---

### 13.2 Kubernetes Application Access Example

If the business runs on Kubernetes, use Secret to store credentials.

Example:

    apiVersion: v1
    kind: Secret
    metadata:
      name: app-rustfs-secret
      namespace: app
    type: Opaque
    stringData:
      OBJECT_STORAGE_ACCESS_KEY: "<business AccessKey>"
      OBJECT_STORAGE_SECRET_KEY: "<business SecretKey>"

ConfigMap stores non-sensitive configuration:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app-rustfs-config
      namespace: app
    data:
      OBJECT_STORAGE_TYPE: "s3"
      OBJECT_STORAGE_ENDPOINT: "https://s3.rustfs.local"
      OBJECT_STORAGE_REGION: "us-east-1"
      OBJECT_STORAGE_BUCKET: "app-uploads"
      OBJECT_STORAGE_FORCE_PATH_STYLE: "true"
      OBJECT_STORAGE_USE_SSL: "true"

Deployment reference:

    envFrom:
      - configMapRef:
          name: app-rustfs-config
      - secretRef:
          name: app-rustfs-secret

Security Reminder:

    Secret is not absolutely safe.
    Needs to be combined with RBAC, namespace isolation, auditing, and minimal permissions.
    Do not give administrator keys to regular applications.

---

### 13.3 VM Application Access Example

On VM, use environment variable files:

    mkdir -p /etc/app
    touch /etc/app/object-storage.env
    chmod 600 /etc/app/object-storage.env

Write:

    cat > /etc/app/object-storage.env <<'EOF'
    OBJECT_STORAGE_TYPE=s3
    OBJECT_STORAGE_ENDPOINT=https://s3.rustfs.local
    OBJECT_STORAGE_REGION=us-east-1
    OBJECT_STORAGE_BUCKET=app-uploads
    OBJECT_STORAGE_FORCE_PATH_STYLE=true
    OBJECT_STORAGE_USE_SSL=true
    OBJECT_STORAGE_ACCESS_KEY=<business AccessKey>
    OBJECT_STORAGE_SECRET_KEY=<business SecretKey>
    EOF

Reference in systemd:

    EnvironmentFile=/etc/app/object-storage.env

Security Reminder:

    File permissions must be restricted.
    Do not allow regular users to read keys.
    Do not print SecretKey in logs.

---

## FourteenI don't know.SDK Access Verification Approach

### 14.1 Why can't we only verify mc

mc success can only indicate:

    RustFS basic S3 operations are available.
    mc has basic compatibility with RustFS.

But real business may also use:

    Java AWS SDK
    Python boto3
    Node.js AWS SDK
    Go AWS SDK
    Rust SDK
    MinIO SDK
    Backup software built-in S3 client
    Log system built-in S3 client

Therefore, verification of the actual business SDK must be done before production.

---

### 14.2 SDK Verification Checklist

At least verify:

    Initialize client.
    Create Bucket.
    Check if Bucket exists.
    Upload small file.
    Upload large file.
    Download object.
    Delete object.
    ListObjects.
    HeadObject.
    Presigned URL.
    Multipart Upload.
    Timeout and retry.
    Error code handling.
    Path-style.
    HTTPS certificate trust.
    Concurrent upload/download.

---

### 14.3 Python boto3 Verification Example

Install:

    apt update
    apt install -y python3-pip
    pip3 install boto3

Create test script:

    cat > /tmp/test-rustfs-boto3.py <<'EOF'
    import boto3
    from botocore.client import Config

    endpoint = "http://s3.rustfs.local:9000"
    access_key = "rustfsadmin"
    secret_key = "RustFSAdmin@123456"
    bucket = "sdk-demo"
    key = "python-boto3/hello.txt"

```python
s3 = boto3.client(
    "s3",
    endpoint_url=endpoint,
    aws_access_key_id=access_key,
    aws_secret_access_key=secret_key,
    region_name="us-east-1",
    config=Config(s3={"addressing_style": "path"}),
)

try:
    s3.create_bucket(Bucket=bucket)
except Exception:
    pass

s3.put_object(Bucket=bucket, Key=key, Body=b"hello rustfs from boto3\n")

obj = s3.get_object(Bucket=bucket, Key=key)
print(obj["Body"].read().decode())

resp = s3.list_objects_v2(Bucket=bucket, Prefix="python-boto3/")
for item in resp.get("Contents", []):
    print(item["Key"], item["Size"])
EOF
```

**Execution:**

```bash
python3 /tmp/test-rustfs-boto3.py
```

**Expected Output:**

```
hello rustfs from boto3
python-boto3/hello.txt
```

**Notes:**

This is a basic SDK validation.
Production requires using actual business code, actual SDK versions, and actual permission policies for verification.

---

### 14.4 Node.js AWS SDK v3 Validation Approach

For business using Node.js, common configuration focus points:

```
endpoint
region
credentials
forcePathStyle: true
```

**Validation Content:**

```
PutObjectCommand
GetObjectCommand
ListObjectsV2Command
DeleteObjectCommand
getSignedUrl
```

**Notes:**

This article does not treat Node.js as the main focus.
Here, only remind operations to confirm SDK configurations with development.
If SDK defaults to using Virtual-hosted-style, forcePathStyle may need to be enabled.

---

## Fifteen, Large Objects and Multipart Upload Validation

### 15.1 Why Validate Large Objects

Common large objects in object storage:

```
Database backup packages
Image tarballs
Model files
Video files
Log archive packages
Dataset compressed packages
```

Large object uploads involve:

```
Network stability.
Client timeout.
Reverse proxy timeout.
Nginx body size.
Multipart Upload.
Temporary files.
Disk space.
Failure retry.
```

Basic upload of 100Mi does not equal sufficient production large object capability.

---

### 15.2 Create 1Gi Test File

**Execution:**

```bash
dd if=/dev/zero of=/tmp/rustfs-1g.bin bs=1M count=1024
```

**Check:**

```bash
ls -lh /tmp/rustfs-1g.bin
sha256sum /tmp/rustfs-1g.bin > /tmp/rustfs-1g.sha256
```

---

### 15.3 Use mc to Upload 1Gi Object

**Execution:**

```bash
docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  -v /tmp:/tmpdata \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp /tmpdata/rustfs-1g.bin rustfs-entry/app-uploads/large/rustfs-1g.bin
```

**Check:**

```bash
docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  stat rustfs-entry/app-uploads/large/rustfs-1g.bin
```

---

### 15.4 Download and Verify

**Download:**

```bash
docker run --rm \
  -v /data/rustfs-client/mc-config:/root/.mc \
  -v /tmp:/tmpdata \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp rustfs-entry/app-uploads/large/rustfs-1g.bin /tmpdata/rustfs-1g-download.bin
```

**Verify:**

```bash
sha256sum /tmp/rustfs-1g.bin /tmp/rustfs-1g-download.bin
```

---

### 15.5 Troubleshooting Large Object Upload Failures

Focus on checking:

```
Nginx client_max_body_size.
Nginx proxy_request_buffering.
Nginx proxy_read_timeout.
Nginx proxy_send_timeout.
RustFS backend health.
Client network stability.
Sufficient disk capacity.
Container logs.
Multipart Upload compatibility issues.
```

**Commands:**

```bash
tail -100 /var/log/nginx/error.log
docker logs rustfs-cluster --tail=200
df -hT
curl -i http://s3.rustfs.local:9000/health
```

---

## Sixteen, Common Error Troubleshooting

### 16.1 SignatureDoesNotMatch

**Phenomenon:** /think

SignatureDoesNotMatch
The request signature we calculated does not match the signature you provided

Common causes:

    AccessKey error.
    SecretKey error.
    System time is out of sync.
    Region inconsistency.
    Endpoint inconsistency.
    Host header rewritten by Nginx.
    Path-style / Virtual-hosted-style mismatch.
    HTTP / HTTPS entry inconsistency.
    SDK signature version incompatibility.
    Presigned URL expired or rewritten.

Troubleshoot:

    timedatectl
    echo ${AWS_ACCESS_KEY_ID}
    echo ${AWS_DEFAULT_REGION}
    curl -i http://s3.rustfs.local:9000/health
    tail -100 /var/log/nginx/error.log
    docker logs rustfs-cluster --tail=200

Handling directions:

    Reconfirm AccessKey / SecretKey.
    Synchronize client and server time.
    Test with us-east-1.
    Enable path-style in SDK.
    Nginx preserves Host header.
    Confirm generated Presigned URL Endpoint matches access Endpoint.

---

### 16.2 AccessDenied

Phenomenon:

    AccessDenied
    Forbidden
    403

Common causes:

    Key lacks permissions.
    Bucket does not exist.
    Object does not exist but returns permission error.
    Current account cannot access the Bucket.
    Current account cannot perform PutObject / GetObject / DeleteObject.
    Used incorrect alias.
    Production permission policy restrictions.

Troubleshoot:

    mc alias list
    mc ls rustfs-entry
    mc ls rustfs-entry/app-uploads
    aws --endpoint-url http://s3.rustfs.local:9000 s3 ls

Handling directions:

    Confirm account.
    Confirm Bucket.
    Confirm permission policy.
    Do not use admin account to mask permission issues.
    Production should perform minimal permission testing for business accounts.

---

### 16.3 EndpointConnectionError

Phenomenon:

    Could not connect to the endpoint URL

Common causes:

    Endpoint written incorrectly.
    Domain cannot be resolved.
    Port not open.
    RustFS service not running.
    Nginx not running.
    Firewall interception.
    Using HTTPS but service only supports HTTP.
    Certificate error causing connection failure.

Troubleshoot:

    getent hosts s3.rustfs.local
    ping -c 3 s3.rustfs.local
    nc -vz s3.rustfs.local 9000
    curl -i http://s3.rustfs.local:9000/health
    systemctl status nginx --no-pager
    docker ps | grep rustfs

---

### 16.4 NoSuchBucket

Phenomenon:

    NoSuchBucket

Cause:

    Bucket does not exist.
    Accessed environment is incorrect.
    Alias points to wrong RustFS cluster.
    Bucket name spelling error.
    Deleted but not recreated.

Troubleshoot:

    mc ls rustfs-entry
    aws --endpoint-url http://s3.rustfs.local:9000 s3 ls

Handling:

    mc mb rustfs-entry/<bucket-name>

Or:

    aws --endpoint-url http://s3.rustfs.local:9000 s3 mb s3://<bucket-name>

---

### 16.5 NoSuchKey

Phenomenon:

    NoSuchKey

Cause:

    Object Key does not exist.
    Prefix written incorrectly.
    Case inconsistency.
    Uploaded to other Bucket.
    Uploaded to other environment.
    Deleted but not restored.

Troubleshoot:

    mc ls --recursive rustfs-entry/app-uploads | grep <keyword>

Or:

    aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 ls s3://app-uploads/ --recursive

---

### 16.6 Certificate error

May occur in HTTPS scenarios:

    certificate verify failed
    x509: certificate signed by unknown authority
    SSL validation failed

Cause:

    Self-signed certificate not trusted by client.
    Certificate domain mismatch.
    Certificate expired.
    Client accesses IP but certificate is issued for domain.
    Incomplete intermediate certificate chain.

Handling:

    Use trusted certificate.
    Client trusts enterprise CA.
    Access with correct domain.
    Check certificate validity period.
    mc experiments can temporarily use --insecure, but not recommended for production.

---

## Seventeen, Pre-production Access Verification Checklist

### 17.1 Basic functions

| Check item | Pass |
|---|---|
| mc alias set |  |
| mc ls |  |
| mc mb |  |
| mc cp upload |  |
| mc cp download |  |
| mc stat |  |
| mc rm |  |
| AWS CLI s3 ls |  |
| AWS CLI s3 cp |  |
| AWS CLI presign |  |
| sha256sum verification |  |

---

### 17.2 SDK compatibility

| Check item | Pass |
|---|---|
| Python boto3 |  |
| Java AWS SDK |  |
| Node.js AWS SDK |  |
| Go AWS SDK |  |
| Path-style |  |
| Virtual-hosted-style |  |
| Presigned URL |  |
| Multipart Upload |  |
| Timeout retry |  |
| Error code handling |  |

### 17.3 Access Entry

| Check Item | Pass |
|---|---|
| Internal HTTP |  |
| External HTTPS |  |
| Nginx Host Passthrough |  |
| Nginx Large File Upload |  |
| Nginx Timeout |  |
| Certificate Validity |  |
| Client Trust Certificate |  |
| Backend Nodes Not Exposed to Public Internet |  |

---

### 17.4 Security

| Check Item | Pass |
|---|---|
| Not Use Admin Account for Business |  |
| Business Independent AccessKey |  |
| SecretKey Not Submitted to Git |  |
| Minimum Privilege |  |
| Deletion Permissions Controlled |  |
| Presigned URL Validity Reasonable |  |
| Management Entry Source Restricted |  |
| Access Log Retention |  |

---

## EighteenI don't know.Production Access Template

### 18.1 Access Information Template for Developers

Business Name:

    <Business Name>

Environment:

    dev / test / prod

Object Storage Type:

    S3 Compatible

Endpoint:

    https://s3.rustfs.local

Region:

    us-east-1

Bucket:

    app-uploads

AccessKey:

    <Provided via Secure Channel>

SecretKey:

    <Provided via Secure Channel>

Path-style:

    true

HTTPS:

    true

Purpose:

    User Attachment Upload
    Image Upload
    Backup Package Upload
    Log Archiving

Permission Scope:

    Read/Write in app-uploads Bucket
    No Delete Bucket Permission Granted
    Delete Object Permission Evaluated by Business Needs

Notes:

    Do Not Print SecretKey in Logs.
    Do Not Commit Keys to Git.
    Confirm Multipart Upload Configuration Before Large File Upload.
    Production Must Use HTTPS.
    Presigned URL Validity Should Not Be Too Long.

---

### 18.2 Operations Access Record Template

| Item | Content |
|---|---|
| Business Name |  |
| Environment |  |
| Bucket |  |
| Endpoint |  |
| AccessKey Ownership |  |
| Permission Scope |  |
| Allow Delete Objects |  |
| Allow Delete Bucket | No |
| Use Presigned URL |  |
| Use Multipart Upload |  |
| SDK Language |  |
| SDK Version |  |
| Path-style | Yes / No |
| HTTPS | Yes / No |
| Certificate Source |  |
| Access Time |  |
| Responsible Person |  |
| Rollback Plan |  |

---

## NineteenI don't know.High-Risk Operations Reminder

The following operations must be handled with caution in production environments:

    mc rm --recursive --force
    mc rb
    aws s3 rm --recursive
    Modify Application Endpoint
    Modify AccessKey / SecretKey
    Delete Bucket
    Batch Delete Prefix
    Generate Long-Lived Presigned URL
    Provide Admin Key to Business
    Write SecretKey to Git
    Expose Public Internet via HTTP
    Nginx Reverse Proxy Without Upload Size and Timeout Limits
    Switch Production Traffic Without Verifying SDK

Confirm Before Execution:

    Is it a production environment?
    Which business does the Bucket belong to?
    Are there backups for deleted objects?
    Is there approval?
    Is there a recovery plan?
    Are there access logs?
    Is there a rollback plan?

---

## TwentyI don't know.Experiment Cleanup

### 20.1 Delete Test Objects

Delete test objects under app-uploads:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm --recursive --force rustfs-entry/app-uploads/demo-mirror

Delete awscli-demo:

    aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 rm s3://awscli-demo/ --recursive

---

### 20.2 Delete Test Bucket

Delete empty Bucket:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rb rustfs-entry/awscli-demo

If the Bucket is not empty, clean up objects first.

High-Risk Reminder:

    Production Environment Prohibits Arbitrary Bucket Deletion.

---

### 20.3 Delete Local Test Files

Execute:

    rm -rf /tmp/rustfs-client-demo
    rm -rf /tmp/rustfs-client-download
    rm -rf /tmp/rustfs-mirror-download
    rm -f /tmp/awscli-hello.txt
    rm -f /tmp/awscli-hello-download.txt
    rm -rf /tmp/awscli-dir
    rm -f /tmp/presign-test.txt
    rm -f /tmp/presign-test-download.txt
    rm -f /tmp/rustfs-1g.bin
    rm -f /tmp/rustfs-1g-download.bin
    rm -f /tmp/rustfs-1g.sha256
    rm -f /tmp/test-rustfs-boto3.py

---

### 20.4 Whether to Delete mc Configuration

If continuing with 07 Permission and Security Experiment, recommend retaining:

    /data/rustfs-client/mc-config

If confirmed no longer needed:

    rm -rf /data/rustfs-client/mc-config

---

## Twenty-OneI don't know.Completion Standards for This Article

After completing this article, the following should be at least achieved: /think

| Project | Standard |
|---|---|
| Endpoint Understanding | Clearly distinguish single-node, cluster, and unified entry points |
| mc alias | Successfully configured as rustfs-single or rustfs-entry |
| Bucket | Successfully created app-uploads |
| Object Upload | Successfully upload small files, directories, and large files |
| Object Download | Successfully download |
| Data Validation | sha256sum consistency |
| mc stat | Ability to view object metadata |
| AWS CLI | s3 ls / cp functionality available |
| Presigned URL | Ability to generate and download via curl |
| Path-style | Ability to explain access format |
| Virtual-hosted-style | Ability to explain production complexity |
| Application Configuration | Ability to provide Endpoint, Bucket, Region, and Path-style parameters |
| SDK Awareness | Clearly verify actual business SDK |
| Troubleshooting Ability | Ability to troubleshoot SignatureDoesNotMatch, AccessDenied, and Endpoint errors |
| Security Awareness | Clearly understand SecretKey not submitted to Git, production must use HTTPS |

---

## 22. Interview Answer Approach

If asked:

    How to integrate RustFS client? What parameters does the application need to configure?

Respond:

    RustFS is an S3-compatible object storage, so client integration is similar to MinIO and AWS S3. Applications typically need to configure Endpoint, AccessKey, SecretKey, Region, Bucket, and whether Path-style is enabled. Experimental environments can use http://s3.rustfs.local:9000, the production environment should be used https://s3.rustfs.local as a unified HTTPS entry point.
    I would first validate basic access with mc, for example, mc alias set to configure RustFS endpoint, then create Bucket, upload objects, download objects, and verify file consistency with sha256sum. After mc validation, I would use AWS CLI to perform s3 ls, s3 cp, s3 presign operations to further verify S3 compatibility.
    When integrating applications, I would have developers configure parameters like OBJECT_STORAGE_ENDPOINT, OBJECT_STORAGE_BUCKET, OBJECT_STORAGE_ACCESS_KEY, OBJECT_STORAGE_SECRET_KEY, OBJECT_STORAGE_REGION, and OBJECT_STORAGE_FORCE_PATH_STYLE=true. SecretKey should not be written to Git or printed in logs, and production should use business-specific keys instead of administrator keys.
    Path-style and Virtual-hosted-style need to be confirmed in advance. In private object storage, I generally recommend enabling Path-style for applications, which is in the form of https://s3.rustfs.local/bucket/key. Virtual-hosted-style requires wildcard DNS and wildcard certificate, such as bucket.s3.rustfs.local, with higher configuration complexity.
    If the application needs temporary file downloads, Presigned URL can be used. The backend generates a signed URL with expiration time, allowing frontend or users to access within the validity period. In production, Presigned URL validity should not be too long, and the Endpoint generating the URL must be an HTTPS domain accessible by users.
    Troubleshooting SignatureDoesNotMatch typically involves checking AccessKey, SecretKey, time synchronization, Region, Endpoint, Host header, Path-style, and HTTPS/HTTP consistency; AccessDenied requires checking account permissions, Bucket, and Object Key; EndpointConnectionError requires checking DNS, port, firewall, Nginx, and RustFS service status.
    Before production, validation with mc alone is insufficient, as businesses may use Java, Python, Node.js, Go, etc. SDKs. Verification of Multipart Upload, Presigned URL, large object upload, concurrent upload, timeout retry, and SDK error handling is required.

---

## 23. Summary of This Article

This article completes the learning of RustFS client access:

1. Core parameters for RustFS client access include Endpoint, AccessKey, SecretKey, Bucket, and Region.
2. Endpoint can be a single-node, cluster node, or unified entry point.
3. Production recommends applications access only through unified HTTPS Endpoint.
4. Bucket is the top-level space for object storage.
5. Object Key is the object name, not the actual file system path.
6. mc can manage RustFS Bucket and Object.
7. mc alias set is used to configure RustFS access entry.
8. mc mb can create Bucket.
9. mc cp can upload and download Object.
10. mc stat can view object metadata.
11. AWS CLI can access RustFS via --endpoint-url.
12. AWS CLI can perform basic s3 ls, s3 cp, s3 rm operations.
13. Presigned URL can be used for temporary download or upload authorization.
14. Presigned URL validity must be controlled in production.
15. Path-style is more suitable for private object storage experiments and basic integration.
16. Virtual-hosted-style requires DNS and certificate coordination.
17. Application integration requires clear forcePathStyle/path-style configuration.
18. SecretKey should not be submitted to Git.
19. Ordinary business should not use administrator keys.
20. Production must use HTTPS.
21. mc validation does not guarantee full SDK compatibility.
22. Before production, verify actual business SDK, Multipart Upload, Presigned URL, and large object upload.
23. The next article will learn about RustFS permissions and security: access keys, HTTPS, and reverse proxy.

---

## 24. Reference Documents

RustFS official website:

    https://rustfs.com/

RustFS official documentation:

    https://docs.rustfs.com/

RustFS S3 compatibility:

    https://docs.rustfs.com/features/s3-compatibility/

RustFS mc client documentation:

    https://docs.rustfs.com/developer/mc.html

RustFS JavaScript SDK documentation:

    https://docs.rustfs.com/developer/sdk/javascript.html

RustFS other SDK documentation:

    https://docs.rustfs.com/developer/sdk/other.html

RustFS Docker installation documentation:

    https://docs.rustfs.com/installation/docker/

RustFS Nginx reverse proxy configuration:

    https://docs.rustfs.com/integration/nginx-reverse-proxy-configuration/

RustFS TLS configuration:

    https://docs.rustfs.com/integration/tls-configuration/

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

AWS S3 API documentation: /think

AWS S3 Presigned URL Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html

AWS S3 Virtual Hosting Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html

AWS CLI S3 Documentation:

    https://docs.aws.amazon.com/cli/latest/reference/s3/

boto3 S3 Documentation:

    https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html

Nginx Official Documentation:

    https://nginx.org/en/docs/