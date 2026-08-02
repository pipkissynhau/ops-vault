# RustFS Client Access: S3 API, mc Tool, and Application Integration

Recommended Path: 05-Storage/04-RustFS/06-RustFS Client Access: S3 API, mc Tool, and Application Integration.md

Tags: #RustFS #S3 #Object Storage #mc #AWSCLI #SDK #PresignedURL #PathStyle #VirtualHostedStyle #Application Integration #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the sixth in the RustFS series, focusing on how to access RustFS from clients.

Previous topics include:

    01-RustFS Basics: S3-compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Understanding Single-node and Cluster Configurations
    03-RustFS Single-node Deployment Practice: Service Startup, Data Directory Setup, and Access Verification
    04-RustFS Cluster Deployment Practice: Multiple Nodes, Disks, and Access Points
    05-RustFS vs. MinIO: Architectural, Deployment, Ecosystem, and Operations Differences

This article addresses the following key points:

    How to access RustFS from clients
    What an S3 Endpoint is
    How to configure AccessKey and SecretKey
    How to connect mc to RustFS
    How to create a Bucket using mc
    How to upload and download Objects with mc
    How to access RustFS using AWS CLI
    How to integrate applications with RustFS
    The differences between Path-style and Virtual-hosted-style access
    What a Presigned URL is
    How to generate temporary download links
    Key considerations for uploading large Objects
    Why Multipart Upload requires verification
    How to troubleshoot SignatureDoesNotMatch errors
    How to resolve AccessDenied issues
    How to diagnose Endpoint connection errors
    How to provide a RustFS integration parameter template for businesses

This document assumes that RustFS has already been deployed.

You can use a single-node RustFS at:

    http://10.0.0.51:9000

Or, you can use a cluster with a unified access point at:

    http://s3.rustfs.local:9000

For production environments, it is recommended to use:

    https://s3.rustfs.local

Details on HTTPS certificates, reverse proxies, and security measures will be covered in Section 07.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the RustFS S3 client access model.
2. Comprehend the roles of Endpoint, Bucket, Object Key, AccessKey, SecretKey, and Region.
3. Configure RustFS aliases using mc.
4. Create Buckets using mc.
5. Upload objects using mc.
6. Download objects using mc.
7. View object metadata using mc.
8. Delete objects and Buckets using mc.
9. Access RustFS using AWS CLI.
10. Upload and download objects using AWS CLI.
11. Generate Presigned URLs.
12. Verify Presigned URLs using curl.
13. Differentiate between Path-style and Virtual-hosted-style access.
14. Configure S3 Endpoints for applications.
15. Identify the required parameters for integrating applications with RustFS.
16. Troubleshoot SignatureDoesNotMatch errors.
17. Resolve AccessDenied issues.
18. Diagnose EndpointConnectionError problems.
19. Understand the differences in validation processes between mc, AWS CLI, and SDK clients.
20. Prepare a client compatibility verification checklist before production integration.

---

## III. RustFS Client Access Model

### 3.1 Required Parameters for Client Access

To access RustFS, the following parameters are typically required:

| Parameter | Description | Example |
|--------------|---------------|-----------|
| Endpoint     | RustFS S3 API address | http://10.0.0.51:9000 |
| AccessKey    | Access account       | rustfsadmin |
| SecretKey   | Access secret key    | RustFSAdmin@123456 |
| Bucket        | Object storage bucket  | app-uploads |
| Object Key    | Object name         | images/avatar-001.png |
| Region      | Geographic region     | us-east-1 |
| Path-style    | Access path format     | http://endpoint/bucket/key |
| HTTPS       | Whether to use TLS     | Required for production |

Example parameters:

    Endpoint: http://10.0.0.51:9000
    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456
    Region: us-east-1
    Bucket: app-uploads

Example for a cluster unified access point:

    Endpoint: http://s3.rustfs.local:9000
    AccessKey: rustVerify the compatibility of S3 command standards.
Ensure that the application side follows an access method closer to AWS S3.
Generate Presigned URLs.

The SDK is used for:

Verifying the actual business integration capability.
It cannot be judged solely based on the availability of the mc.

---

## Section 5: Pre-checks Before Client Access

### 5.1 Check the RustFS Service

Execute the following on the RustFS node:

    docker ps | grep rustfs

View the logs:

    docker logs rustfs-single --tail=100

Or for cluster mode:

    docker logs rustfs-cluster --tail=100

---

### 5.2 Check the API Health Status

For a single machine:

    curl -i http://10.0.0.51:9000/health

For the cluster's unified entrance:

    curl -i http://s3.rustfs.local:9000/health

Expected response:

    HTTP/1.1 200 OK

---

### 5.3 Check the Network Connection Between the Client and RustFS

Execute the following on the rustfs-client:

    ping -c 3 10.0.0.51
    ping -c 3 s3.rustfs.local

Check the ports:

    nc -vz 10.0.0.51 9000
    nc -vz s3.rustfs.local 9000

If nc is not available:

    apt update
    apt install -y netcat-openbsd

---

### 5.4 Check Time Synchronization

Execute the following on both the client and RustFS nodes:

    timedatectl

Pay special attention to:

    System clock synchronized: yes

Note:

S3 signature generation depends on the time. A significant time difference may result in a SignatureDoesNotMatch error. It is essential that all clients, servers, and reverse proxy nodes maintain synchronized time.

---

## Section 6: Basic Usage of the mc Client

### 6.1 Prepare the mc Configuration Directory

Execute the following on the client node:

    mkdir -p /data/rustfs-client/mc-config

Test the mc image:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

Note:

This document continues to use the mc image that has already been synchronized from the MinIO module. The mc client can access both MinIO and S3-compatible object storage systems like RustFS.

---

### 6.2 Configure an Alias for a Single Machine RustFS

If you need to access a single-machine RustFS:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-single http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@123456'

View the aliases:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 6.3 Configure an Alias for the Cluster's Unified Entrance

If you need to access the cluster's unified entrance:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-entry http://s3.rustfs.local:9000 rustfsadmin 'RustFSAdmin@123456'

View the aliases:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 6.4 View the Bucket List

For a single machine:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-sProduction must be approved.
Before deletion, it is essential to confirm that the bucket is empty or that the data has been backed up.
Before deletion, make sure no services are currently using it.

---

## Section 8: MC Object Upload and Download

### 8.1 Prepare Test Files

Execute on the client node:

    mkdir -p /tmp/rustfs-client-demo
    cd /tmp/rustfs-client-demo

Create a text file:

    echo "hello rustfs client access" > hello.txt

Create a configuration file:

    cat > app.conf <<'EOF'
    app_name=rustfs-client-demo
    env=lab
    endpoint=http://s3.rustfs.local:9000
    bucket=app-uploads
    EOF

Create a logs directory:

    mkdir -p logs
    echo "2026-04-28 INFO rustfs client upload test" > logs/app.log

Create a JSON file:

    mkdir -p data
    cat > data/user.json <<'EOF'
    {"id":1001,"name":"ops-user","storage":"rustfs"}
    EOF

Create a large file:

    dd if=/dev/zero of=file-100m.bin bs=1M count=100

Calculate the checksums:

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

Check the result:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-entry/app-uploads

---

### 8.3 Upload a Directory

Upload the logs directory:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-client-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /demo/logs rustfs-entry/app-uploads/

Check the result:

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

Check the result:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-entry/app-uploads

---

### 8.5 Upload a Large File

Upload the 100Mi file:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-client-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-entry/app-uploads

---

### 9.2 Viewing Object Details

Viewing object metadata:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      stat rustfs-entry/app-uploads/hello.txt

Viewing large files:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      stat rustfs-entry/app-uploads/file-100m.bin

---

### 9.3 Deleting a Single Object

Deleting an object:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm rustfs-entry/app-uploads/hello.txt

Checking the deletion:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-entry/app-uploads

High-risk warning:

    Deleting an object is an irreversible operation.
    Make sure to back up the object before deleting it in production.
    Do not grant ordinary business accounts the permission to delete objects.

---

### 9.4 Removing Directory Prefixes

Removing the "logs" prefix:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm --recursive --force rustfs-entry/app-uploads/logs

High-risk warning:

    "Directories" in object storage are essentially Key prefixes.
    Removing prefixes may result in the deletion of many objects.
    Proceed with caution in production.

---

### 9.5 Synchronizing Objects Using Mirroring

Synchronizing a local directory to RustFS:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-client-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mirror /demo rustfs-entry/app-uploads/demo-mirror

Synchronizing from RustFS to local:

    mkdir -p /tmp/rustfs-mirror-download

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      -v /tmp/rustfs-mirror-download:/download \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mirror rustfs-entry/app-uploads/demo-mirror /download

Note:

    The "mirror" command is commonly used for migration and synchronization.
    Always understand the behavior of overwriting, deleting, and incremental synchronization before using it in production.
    Do not directly execute unfamiliar "mirror" commands on production Buckets.

---

## Section X: Accessing RustFS Using AWS CLI

### 10.1 Installing AWS CLI

For Ubuntu:

    apt update
    apt install -y awscli

Checking the version:

    aws --version

Note:

    The awscli version in the apt repository may be outdated.
    Basic commands like s3 ls, cp, and presign can be used for testing purposes.
    For production use, make sure to verify compatibility with the actual version of the SDK.

---

### 10.2 Setting Environment Variables

Execute the following commands:

    export AWS_ACCESS_KEY_ID="rustfsadmin"
    export AWS_SECRET_ACCESS_KEY="RustFSAdmin@123456"
    export AWS_DEFAULT_REGION="us-east-1"

Check the settings:

    echo ${AWS_ACCESS_KEY_ID}
    echo ${AWS_DEFAULT_REGION}

The URL is valid for a specified period of time. It is commonly used for temporary downloads, temporary uploads, file sharing, and authorized access to private objects.

**Typical scenarios:**

- Users download private attachments.
- Users temporarily upload files.
- The backend generates a URL that the frontend can directly access.
- This reduces the burden on application servers when transferring large files.

**Notes:**

- A Presigned URL contains signature parameters.
- If the URL is leaked, it may still be accessed within its validity period.
- The validity period should not be too long.
- In production environments, measures such as permissions, expiration times, download restrictions, and log auditing should be implemented in conjunction with Presigned URLs.

---

### 11.2 Preparing an Object

Let's upload a test object:

```bash
echo "presigned url test" > /tmp/presign-test.txt
aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 cp /tmp/presign-test.txt s3://awscli-demo/presign-test.txt
```

To check the object, run:

```bash
aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 ls s3://awscli-demo/presign-test.txt
```

---

### 11.3 Generating a Download Presigned URL

Generate a URL that is valid for 10 minutes:

```bash
aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 presign s3://awscli-demo/presign-test.txt \
      --expires-in 600
```

Save the output:

```bash
PRESIGNED_URL=$(aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 presign s3://awscli-demo/presign-test.txt \
      --expires-in 600)
echo "${PRESIGNED_URL}"
```

---

### 11.4 Using curl to Download

Execute the following command:

```bash
curl -L "${PRESIGNED_URL}" -o /tmp/presign-test-download.txt
```

Check the downloaded file:

```bash
cat /tmp/presign-test-download.txt
```

Verify the integrity of the file:

```bash
sha256sum /tmp/presign-test.txt /tmp/presign-test-download.txt
```

---

### 11.5 Common Issues with Presigned URLs

If access fails, check the following:

- Whether the URL has expired.
- Whether the client's time is accurate.
- Whether the RustFS node's time is accurate.
- Whether the Endpoint is accessible.
- Whether the domain name in the URL can be resolved by the client.
- Whether HTTP or HTTPS is being used.
- Whether the reverse proxy is modifying the Host header.
- Whether the Bucket and Object Key are correct.
- Whether the credentials used to generate the URL have the necessary GetObject permissions.

**Production notes:**

- If an application generates Presigned URLs for external use, the Endpoint must be a publicly accessible domain name.
- Do not provide private network addresses like http://10.0.0.51:9000 to public users.
- In production, generate accessible addresses such as https://s3.example.com/bucket/key.

---

## Chapter 12: Path-style and Virtual-hosted-style Access

### 12.1 Path-style Access

**Format:**

```bash
http://s3.rustfs.local:9000/app-uploads/hello.txt
```

**Structure:**

- Endpoint: http://s3.rustfs.local:9000
- Bucket: app-uploads
- Object: hello.txt

**Full path:**

/app-uploads/hello.txt

**Advantages:**

- Simple to experiment with.
- No need to configure DNS for each Bucket.
- Easier to verify compatibility with private object storage solutions.
- Relatively simple to configure in Nginx.

---

### 12.2 Virtual-hosted-style Access

**Format:**

```bash
http://app-uploads.s3.rustfs.local:9000/hello.txt
```

**Structure:**

- Bucket: app-uploads
- Endpoint: s3.rustfs.local
- Object: hello.txt

**Full domain name:**

app-uploads.s3.rustfs.local

**Advantages:**

- More consistent with the access style used by AWS S3.
- Suitable for scenarios where some SDKs have default configurations that rely on this style.
- Better suited for production environments with more detailed domain naming conventions.

**Challenges:**

- Requires wildcard DNS configuration.
- Requires wildcard certificates.
- Nginx needs to correctly forward the Host header.
- Configuration complexity increases in private environments.

---

### 12.3 Experimental Recommendations

During```markdown
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

Security reminder:

    Secrets are not absolutely secure.
    They should be used in conjunction with RBAC, namespace isolation, auditing, and minimal privilege principles.
    Never provide administrative keys to ordinary applications.

---

### 13.3 Example of VM Application Access

On a virtual machine, you can use an environment variable file:

    mkdir -p /etc/app
    touch /etc/app/object-storage.env
    chmod 600 /etc/app/object-storage.env

Write the following content to the file:

    cat > /etc/app/object-storage.env <<'EOF'
    OBJECT_STORAGE_TYPE=s3
    OBJECT_STORAGE_ENDPOINT=https://s3.rustfs.local
    OBJECT_STORAGE_REGION=us-east-1
    OBJECTStorage_BUCKET=app-uploads
    OBJECT_STORAGE_FORCE_PATH_STYLE=true
    OBJECT_STORAGE_USE_SSL=true
    OBJECT_STORAGE_ACCESS_KEY=<Business Access Key>
    OBJECT_STORAGE_SECRET_KEY=<Business Secret Key>
    EOF

Reference in systemd:

    EnvironmentFile=/etc/app/object-storage.env

Security reminder:

    File permissions must be restricted.
    Ordinary users should not have access to the secret keys.
    The SecretKey should never be printed in logs.

---

## Section 14: SDK Access Verification Approach

### 14.1 Why Can't We Just Verify mc?

A successful test with mc only confirms that:

    The basic S3 operations of RustFS are functional.
    There is basic compatibility between mc and RustFS.

However, in real-world applications, other SDKs might also be used, such as:

    Java AWS SDK
    Python boto3
    Node.js AWS SDK
    Go AWS SDK
    Rust SDK
    MinIO SDK
    Built-in S3 clients in backup software
    Built-in S3 clients in logging systems

Therefore, it is essential to verify the actual SDKs used in production before deployment.

---

### 14.2 SDK Verification Checklist

At a minimum, the following functions should be verified:

    Initializing the client.
    Creating a bucket.
    Checking if the bucket exists.
    Uploading small files.
    Uploading large files.
    Downloading objects.
    Deleting objects.
    Listing objects.
    Getting object metadata.
    Generating presigned URLs.
    Multipart uploading.
    Handling timeouts and retries.
    Error code handling.
    Path-style functionality.
    HTTPS certificate validation.
    Concurrent uploads and downloads.

---

### 14.3 Example of Python boto3 Verification

Installation:

    apt update
    apt install -y python3-pip
    pip3 install boto3

Create a test script:

    cat > /tmp/test-rustfs-boto3.py <<'EOF'
    import boto3
    from botocore.client import Config

    endpoint = "http://s3.rustfs.local:9000"
    access_key = "rustfsadmin"
    secret_key = "RustFSAdmin@123456"
    bucket = "sdk-demo"
    key = "python-boto3/hello.txt"

    s3 = boto3.client(
        "s3",
        endpoint_url=endpoint,
        aws_access_key_id=access_key,
        aws_secret_access_key=secret_key,
        region_name="us-east-1",
        config=Config(s3={"addressing_style": "path"},
    )

    try:
        s3.create_bucket(Bucket=bucket)
    except Exception:
        pass

    s3.put_object(Bucket=bucket, Key=key, Body=b"hello rustfs from boto3\n")

    obj = s3.get_object(Bucket=bucket, Key ключ)
    print(obj["Body"].read().decode())

    resp = s3.list_objects_v2(Bucket=bucket, Prefix="python-boto3/")
    for item in resp.get("Contents", []):
        print(item["Key"], item["Size"])
    EOF

Execution:

    python3 /tmp/test-rustfs-boto3.py

Expected results:

    "hello rustfs from boto3"
    "python-boto3/hello.txt"

Note:

    This is just a basic SDK verification. In production, you should use the actual code, SDK versions, and permission settings specific to your application.

---

### 14.4 Verification Approach for Node.js AWS SDK v3

For Node.js applications, key configuration items include:

    endpoint
    region
    credentials
    forcePathStyle: true

Verification tasks include:

    Testing PutObjectCommand.
    Testing GetObjectCommand.
    Testing List      cp rustfs-entry/app-uploads/large/rustfs-1g.bin /tmpdata/rustfs-1g-download.bin

Verification:

    sha256sum /tmp/rustfs-1g.bin /tmp/rustfs-1g-download.bin

---

### 15.5 Troubleshooting for Large Object Upload Failures

Key areas to check:

    Nginx's client_max_body_size setting.
    Nginx's proxy_request_buffering settings.
    Nginx's proxy_read_timeout and proxy_send_timeout settings.
    Whether the RustFS backend is running properly.
    The stability of the client's network connection.
    Whether there is sufficient disk space.
    Any abnormal container logs.
    Possibility of compatibility issues with Multipart Upload.

Commands to use:

    tail -100 /var/log/nginx/error.log
    docker logs rustfs-cluster --tail=200
    df -hT
    curl -i http://s3.rustfs.local:9000/health

---

## Chapter Sixteen: Common Error Troubleshooting

### 16.1 SignatureDoesNotMatch

Symptom:

    SignatureDoesNotMatch
    The signature calculated by our system does not match the one provided by you.

Common causes:

    Incorrect AccessKey.
    Incorrect SecretKey.
    Out-of-sync system time.
    Different Regions or Endpoints.
    Nginx modifying the Host header.
    Mismatch between path-style and virtual-hosted-style access.
    Inconsistent HTTP/HTTPS endpoints.
    Incompatible SDK signature versions.
    Expired or modified Presigned URL.

Troubleshooting steps:

    Check timedatectl.
    Verify ${AWS_ACCESS_KEY_ID} and ${AWS_DEFAULT_REGION}.
    Run curl -i http://s3.rustfs.local:9000/health.
    Review the last 100 lines of /var/log/nginx/error.log.
    Check docker logs rustfs-cluster --tail=200.

Action plans:

    Reconfirm the AccessKey and SecretKey.
    Ensure synchronization of time between the client and server.
    Test using us-east-1 region.
    Enable path-style access in the SDK.
    Make sure Nginx retains the Host header.
    Verify that the Presigned URL's endpoint matches the actual access endpoint.

---

### 16.2 AccessDenied

Symptom:

    AccessDenied
    Forbidden
    403 Error

Common causes:

    Insufficient permissions for the key.
    The Bucket does not exist.
    The Object does not exist, but an access error is reported.
    The current account does not have permission to access this Bucket.
    The current account lacks the necessary rights to perform PutObject, GetObject, or DeleteObject operations.
    Using the wrong alias.
    Restrictions imposed by the production permission policy.

Troubleshooting steps:

    List all aliases using mc alias list.
    Verify the contents of the rustfs-entry and app-uploads folders using mc ls.
    Use aws --endpoint-url http://s3.rustfs.local:9000 s3 ls to check the Bucket.

Action plans:

    Confirm the account details.
    Verify the existence of the Bucket.
    Check the permission policy.
    Avoid using an administrative account to resolve permission issues.
    Ensure that business accounts have minimal necessary permissions for production use.

---

### 16.3 EndpointConnectionError

Symptom:

    Unable to connect to the specified endpoint URL.

Common causes:

    Incorrect endpoint address.
    Domain name cannot be resolved.
    Port is unreachable.
    The RustFS service is not running.
    Nginx is not running.
    Firewalls are blocking the connection.
    Using HTTPS while the service only supports HTTP.
    Certificate errors preventing connection.

Troubleshooting steps:

    Use getent hosts to check the IP address of s3.rustfs.local.
    Try ping -c 3 s3.rustfs.local to test connectivity.
    Use nc -vz s3.rustfs.local 9000 to check port availability.
    Verify the status of Nginx using systemctl status nginx --no-pager.
    Check if any RustFS containers are running using docker ps | grep rustfs.

---

### 16.4 NoSuchBucket

Symptom:

    NoSuchBucket error is received.

Possible causes:

    The Bucket does not exist.
    Incorrect access environment settings.
    The alias points to the wrong RustFS cluster.
    Spelling mistake in the Bucket name.
    The Bucket was deleted and not recreated.

Troubleshooting steps:

    List the contents of the rustfs-entry folder using mc ls.
    Use aws --endpoint-url http://s3.rustfs.local:9000 s3 ls to check theBucket.

Action plans:

    Create the Bucket if it does not exist using mc mb rustfs-entry/<bucket-name>.
    Or, use aws --endpoint-url http://s3.rustDo not print the SecretKey in the logs.
Do not submit the key to Git.
Confirm the multipart upload configuration before uploading large files.
HTTPS must be used in production environments.
The validity period of a Presigned URL should not be too long.

---

### 18.2 Operations and Maintenance Access Record Template

| Project | Details |
|---|---|
| Business Name |  |
| Environment |  |
| Bucket |  |
| Endpoint |  |
| AccessKey Owner |  |
| Permission Scope |  |
| Whether Object Deletion Is Allowed |  |
| Whether Bucket Deletion Is Allowed | No |
| Whether to Use Presigned URL |  |
| Whether to Use Multipart Upload |  |
| SDK Language |  |
| SDK Version |  |
| Path-style | Yes / No |
| HTTPS | Yes / No |
| Certificate Source |  |
| Access Time |  |
| Responsible Person |  |
| Rollback Plan |  |

---

## Chapter Nineteen: High-Risk Operation Warnings

The following operations must be carried out with caution in a production environment:

    mc rm --recursive --force
    mc rb
    aws s3 rm --recursive
    Modifying the application's Endpoint
    Changing AccessKey / SecretKey
    Deleting a Bucket
    Batch deleting Prefixes
    Generating long-term valid Presigned URLs
    Allowing business users to access administrative keys
    Saving SecretKeys in Git
    Exposing services over HTTP
    Not restricting upload size and timeout settings for Nginx reverse proxies
    Switching production traffic without verifying the SDK

Before executing any of these operations, it is essential to confirm:

    Whether it is a production environment.
    Which business the Bucket belongs to.
    Whether there are backups for deleted objects.
    Whether approval has been obtained.
    Whether a recovery plan is in place.
    Whether access logs are being recorded.
    Whether a rollback strategy exists.

---

## Chapter Twenty: Experimental Cleanup

### 20.1 Deleting Test Objects

Delete test objects under app-uploads:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm --recursive --force rustfs-entry/app-uploads/demo-mirror

Delete awscli-demo:

    aws --endpoint-url http://s3.rustfs.local:9000 \
      s3 rm s3://awscli-demo/ --recursive

---

### 20.2 Deleting Test Buckets

Delete empty Buckets:

    docker run --rm \
      -v /data/rustfs-client/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rb rustfs-entry/awscli-demo

If the Bucket is not empty, clear its contents first.

High-Risk Warning:

    Deleting Buckets in a production environment is strictly prohibited.

---

### 20.3 Deleting Local Test Files

Execute the following commands:

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

### 20.4 Deciding Whether to Keep the mc Configuration

If you plan to continue with experiments related to permissions and security in Chapter 07, it is recommended to keep:

    /data/rustfs-client/mc-config

If you confirm that you will no longer use it, then delete it:

    rm -rf /data/rustfs-client/mc-config

---

## Chapter Twenty-One: Standards for Completing This Tutorial

After completing this tutorial, you should have achieved at least the following:

| Project | Standard |
|---|---|
| Understanding of Endpoints | Clearly distinguish between standalone nodes, clusters, and a unified entry point. |
| mc alias Configuration | Successfully configure either rustfs-single or rustfs-entry. |
| Bucket Creation | Successfully create the app-uploads Bucket. |
| Object Upload | Successfully upload small files, directories, and large files. |
| Object8. The mc mb command can be used to create Buckets.
9. The mc cp command can be used to upload and download Objects.
10. The mc stat command can be used to view object metadata.
11. The AWS CLI can access RustFS using the --endpoint-url parameter.
12. The AWS CLI can be used for basic S3 operations such as ls, cp, and rm.
13. Presigned URLs can be used for temporary authorization of downloads or uploads.
14. In production environments, it is essential to control the validity period of Presigned URLs.
15. The path-style is more suitable for private object storage experiments and basic access.
16. The virtual-hosted-style requires coordination with DNS and certificates.
17. When integrating applications, it is necessary to explicitly configure whether to use forcePathStyle or path-style.
18. The SecretKey should not be stored in Git.
19. Regular operations should not use administrative keys.
20. HTTPS must be used in production environments.
21. Just because the mc command works does not mean that the SDK is fully compatible.
23. Before going into production, it is essential to verify that the actual SDK, Multipart Upload, Presigned URLs, and large object upload functionality work correctly.
24. In the next article, we will learn about RustFS permissions and security: access keys, HTTPS, and reverse proxies.

---

## 24. References

RustFS Official Website:

    https://rustfs.com/

RustFS Official Documentation:

    https://docs.rustfs.com/

RustFS S3 Compatibility Documentation:

    https://docs.rustfs.com/features/s3-compatibility/

RustFS mc Client Documentation:

    https://docs.rustfs.com/developer/mc.html

RustFS JavaScript SDK Documentation:

    https://docs.rustfs.com/developer/sdk/javascript.html

RustFS Other SDK Documentation:

    https://docs.rustfs.com/developer/sdk/other.html

RustFS Docker Installation Documentation:

    https://docs.rustfs.com/installation/docker/

RustFS Nginx Reverse Proxy Configuration Documentation:

    https://docs.rustfs.com/integration/nginx-reverse-proxy-configuration/

RustFS TLS Configuration Documentation:

    https://docs.rustfs.com/integration/tls-configuration/

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

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