# Ceph RGW Object Storage: S3 Interface, Bucket, User, and Object Management Practices

Recommended Path: 05-Storage/01-Ceph/11-Ceph RGW Object Storage: S3 Interface, Bucket, User, and Object Management Practices.md

Tags: #Ceph #RGW #RADOSGateway #ObjectStorage #S3 #Bucket #AccessKey #SecretKey #radosgw-admin #Reverse Proxy #HTTPS #SRE #Distributed Storage #Advanced SRE

---

## I. Document Overview

This article is the eleventh in the Ceph Advanced SRE storage series, focusing on the theory, practical operations, and troubleshooting methods of Ceph RGW object storage.

Previous topics covered include:

- Ceph infrastructure
- RADOS, MON, MGR, OSD, CRUSH
- Differences between RBD, CephFS, and RGW storage types
- cephadm cluster initialization
- OSD management
- Pool and PG concepts
- CRUSH fault domains
- RBD block storage practices
- CephFS file storage practices

This article delves into the third core usage of Ceph:

    RGW Object Storage

RGW stands for:

    RADOS Gateway

It can be understood as:

    A S3/Swift-compatible object storage gateway provided by Ceph.

RGW is suitable for:

- Image storage
- Attachment storage
- Backup and archiving
- Log archiving
- Private S3 object storage
- Applications accessing files via the S3 SDK
- Business scenarios compatible with cloud object storage interfaces

This article covers the following key areas:

- What RGW is
- The relationship between RGW and RADOS
- Differences between RGW and MinIO
- How to deploy RGW using cephadm
- How to specify RGW nodes and ports
- How to check the status of the RGW service
- How to create S3 users
- How to obtain AccessKey and SecretKey
- How to access RGW using AWS CLI
- How to create Buckets
- How to upload, download, and delete objects
- How to check the status of Buckets
- How to configure user quotas and Bucket quotas
- How to expose a unified entry point through Nginx/LB/HTTPS
- How to troubleshoot S3 access failures, permission issues, port connectivity problems, and Bucket abnormalities
- High availability considerations for RGW in production environments

---

## II. Experimental Objectives

After completing this article, you should be able to:

1. Understand the object storage model of RGW.
2. Comprehend the relationship between Users, Buckets, and Objects.
3. Deploy the RGW service using cephadm.
4. Specify RGW runtime nodes, the number of instances, and listening ports.
5. Check the status of the RGW service and its access ports.
6. Use radosgw-admin to create S3 users.
7. Obtain user AccessKey and SecretKey.
8. Configure an S3 client using AWS CLI.
9. Create Buckets using the RGW Endpoint.
10. Upload, download, view, and delete objects.
11. View Bucket statistics.
12. Set user-level quotas.
13. Set Bucket-level quotas.
14. Understand the automatic creation and role of RGW Pools.
15. Comprehend the high-availability concept behind multiple RGW instances.
16. Understand the distinction between internal HTTP and external HTTPS.
17. Use Nginx/LB to expose a unified HTTPS entry point for RGW.
18. Troubleshoot various issues such as RGW service failures, S3 access errors, permission problems, signature errors, and Bucket anomalies.

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This article uses the same Ceph cluster experimental environment as previous modules.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON/MGR/OSD/RGW |
| 10.0.0.32 | ceph-node02 | MON/MGR/OSD/RGW |
| 10.0.0.33 | ceph-node03 | MON/MGR/OSD/RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion/Fault Testing (optional) |
| 10.0.0.35 | ceph-client | S3 Client Testing (optional) |
| 10.0.0.36 | rgw-lb | Nginx/HAProxy/Unified Entry Point (optional) |

Primary experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

---

### 3.2 RGW Port Planning

This experiment### 13.1 Install AWS CLI on Ceph Client

For Ubuntu:

```bash
apt update
apt install -y awscli jq curl
```

For Rocky Linux 9:

```bash
dnf install -y awscli jq curl
```

Verify installation by running:

```bash
aws --version
jq --version
curl --version
```

### 13.2 Configure AWS CLI Access to RGW

Create an environment variable `AWS_ACCESS_KEY_ID` and `AWS_SECRET_KEY` with your access key and secret key from earlier experiments.

Then, configure the AWS CLI to use these credentials:

```bash
export AWS_ACCESS_KEY_ID=$ACCESS_KEY_ID
export AWS_SECRET_KEY=$SECRET_KEY
aws config set default region us-east-2
aws config set default output format json
```

### 13.3 Verify AWS CLI Configuration

Run the following commands to test the configuration:

```bash
aws s3 get bucket --region us-east-2 my-bucket
aws radosgw info
```

If the commands succeed, it means the AWS CLI is configured correctly to access the RGW service.### 13.1 Configuring the ceph-client Profile

On ceph-client, perform the following command:

    aws configure --profile ceph-rgw

Follow the prompts and enter:

    AWS Access Key ID [None]: The AccessKey generated in the previous step
    AWS Secret Access Key [None]: The SecretKey generated in the previous step
    Default region name [None]: us-east-1
    Default output format [None]: json

---

### 13.2 Configuring Path-style Access

In RGW experiments, IP + port access is used; it is recommended to configure path-style:

    aws configure set profile.ceph-rgw.s3.addressing_style path

To check the configuration:

    cat ~/.aws/config
    cat ~/.aws/credentials

Explanation:

    The path-style format is similar to:
        http://10.0.0.31:7480/bucket/object

    The virtual-host-style format is similar to:
        http://bucket.example.com/object

When using IP addresses in the experimental environment, path-style is more straightforward.

---

### 13.3 Setting the Endpoint Variable

    export RGW_ENDPOINT="http://10.0.0.31:7480"

If you want to test another RGW node:

    export RGW_ENDPOINT="http://10.0.0.32:7480"

If using a unified entry point:

    export RGW_ENDPOINT="https://s3.example.com"

---

### 13.4 Testing Bucket Listing

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

New users may not have any output if they don't have a Bucket.

As long as there are no authentication errors, it indicates that everything is basically working.

---

## Chapter Fourteen: Experiment Six: Creating a Bucket and Uploading an Object

### 14.1 Creating a Bucket

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 mb s3://demo-bucket

Expected output:

    make_bucket: demo-bucket

---

### 14.2 Viewing the Bucket

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

Expected output:

    demo-bucket

---

### 14.3 Creating a Test File

    echo "hello ceph rgw s3" > /tmp/rgw-test.txt

To view the file:

    cat /tmp/rgw-test.txt

---

### 14.4 Uploading an Object

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-test.txt s3://demo-bucket/rgw-test.txt

Expected output:

    upload: ../../tmp/rgw-test.txt to s3://demo-bucket/rgw-test.txt

---

### 14.5 Viewing the Object

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://demo-bucket/

Expected output:

    rgw-test.txt

---

## Chapter Fifteen: Experiment Seven: Downloading and Deleting Objects

### 15.1 Downloading an Object

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp s3://demo-bucket/rgw-test.txt /tmp/rgw-test-download.txt

To view the downloaded file:

    cat /tmp/rgw-test-download.txt

Expected output:

    hello ceph rgw s3

---

### 15.2 Deleting an Object

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rm s3://demo-bucket/rgw-test.txt

To verify the deletion:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://demo-bucket/

Expected output:

    rgw-test.txt should no longer be displayed

---

### 15.3 Uploading a Directory

Create a directory:

    mkdir -p /tmp/rgw-dir
    echo "file1" > /tmp/rgw-dir/file1.txt
    echo "file2" > /tmp/rgw-dir/file2.txt

Upload the directory:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-dir s3://demo-bucket/dir/ --recursive

To check the uploaded files:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://demo-bucket/dir/

Expected output:

    file1## Section Eighteen: Experiment Ten: Exposing a Unified HTTPS Entrance via Nginx

### 18.1 Why a Unified Entrance is Needed

If clients directly access multiple RGW nodes:

    http://10.0.0.31:7480
    http://10.0.0.32:7480
    http://10.0.0.33:7480

Problems include:

- Complex client configuration
- Need to switch nodes in case of failures
- Lack of a unified domain name
- Difficulty in enabling HTTPS
- Inconvenience in accessing access logs and implementing rate limiting
- Challenges in performing grayscale deployments and maintenance

In production, it is recommended to use the following architecture:

    Client
      |
      v
    HTTPS Domain Name / Unified Entrance
      |
      v
    Nginx / HAProxy / LB
      |
      v
    Multiple RGW Backend Instances

---

### 18.2 Nginx Reverse Proxy Topology

Example:

    Client
      |
      | HTTPS 443
      v
    rgw.example.com / 10.0.0.36
      |
      | HTTP 7480
      v
    10.0.0.31:7480
    10.0.0.32:7480
    10.0.0.33:7480

Explanation:

- Exposes HTTPS to the outside world.
- Uses HTTP on the internal RGW backend.
- The internal network must be secure.
- If communicating over an untrusted network, TLS should also be enabled internally.

---

### 18.3 Nginx Upstream Example

Install Nginx on the rgw-lb node:

For Ubuntu:

    apt update
    apt install -y nginx

For Rocky Linux 9:

    dnf install -y nginx

Configuration example:

    cat > /etc/nginx/conf.d/rgw.conf <<'EOF'
    upstream ceph_rgw_backend {
        server 10.0.0.31:7480 max_fails=3 fail_timeout=10s;
        server 10.0.0.32:7480 max_fails=3 fail_timeout=10s;
        server 10.0.0.33:7480 max_fails=3 fail_timeout=10s;
    }

    server {
        listen 80;
        server_name rgw.example.com;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name rgw.example.com;

        ssl_certificate     /etc/nginx/ssl/rgw.example.com.pem;
        ssl_certificate_key /etc/nginx/ssl/rgw.example.com.key;

        client_max_body_size 0;

        proxy_request_buffering off;
        proxy_buffering off;

        location / {
            proxy_pass http://ceph_rgw_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
        }
    }
    EOF

Check the configuration:

    nginx -t

Start the service:

    systemctl enable --now nginx
    systemctl reload nginx

---

### 18.4 Testing with an HTTPS Endpoint

Configure hosts or DNS on the client:

    10.0.0.36 rgw.example.com

Set the endpoint:

    export RGW_ENDPOINT="https://rgw.example.com"

Perform a test:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_endpoint} \
      s3 ls

If using a self-signed certificate, you may need to add:

    --no-verify-ssl

However, this is not recommended for production use.

Production environments should use trusted CA certificates.

---

### 18.5 Production Recommendations for HTTPS

In a production environment, it is advised to:

- Use a formal domain name
- Employ trusted CA certificates
- Expose only port 443 externally
- Do not expose internal RGW ports to the public network
- Configure access logging
- Set limits on the number of connections and request body sizes
- Implement health checks
- Configure rate limiting or WAF based on business requirements
- Ensure that AccessKey / SecretKey are not transmitted over unsecured channels
- Include object storage entries in monitoring and alert systems

---

## Section Nineteen: High Availability Design for RGW

### 19.1 Minimum Experiment

For the experiment, you can use:

    1 RGW instance

Example:

    ceph    radosgw-admin user create --uid="s3-user" --display-name="S3 Test User"
    radosgw-admin user info --uid="s3-user"
    radosgw-admin user list
    radosgw-admin user rm --uid="s3-user"

---

### 21.3 Bucket

    radosgw-admin bucket list
    radosgw-admin bucket list --uid=s3-user
    radosgw-admin bucket stats --bucket=demo-bucket

---

### 21.4 Quotas

    radosgw-admin quota set --quota-scope=user --uid=s3-user --max-size=10G --max-objects=10000
    radosgw-admin quota enable --quota-scope=user --uid=s3-user
    radosgw-admin quota disable --quota-scope=user --uid=s3-user

    radosgw-admin quota set --quota-scope=bucket --uid=s3-user --max-size=5G --max-objects=5000
    radosgw-admin quota enable --quota-scope=bucket --uid=s3-user
    radosgw-admin quota disable --quota-scope=bucket --uid=s3-user

---

### 21.5 AWS CLI

    aws configure --profile ceph-rgw
    aws configure set profile.ceph-rgw.s3.addressing_style path

    aws --profile ceph-rgw --endpoint-url http://10.0.0.31:7480 s3 ls
    aws --profile ceph-rgw --endpoint-url http://10.0.0.31:7480 s3 mb s3://demo-bucket
    aws --profile ceph-rgw --endpoint-url http://10.0.0.31:7480 s3 cp /tmp/file.txt s3://demo-bucket/
    aws --profile ceph-rgw --endpoint-url http://10.0.0.31:7480 s3 ls s3://demo-bucket/
    aws --profile ceph-rgw --endpoint-url http://10.0.0.31:7480 s3 rm s3://demo-bucket/file.txt
    aws --profile ceph-rgw --endpoint-url http://10.0.0.31:7480 s3 rb s3://demo-bucket

---

## Chapter Twenty-Two: Common Issues and Troubleshooting

### 22.1 The RGW Service Is Not Running

To check:

    ceph orch ls --service_type rgw
    ceph orch ps --daemon_type rgw
    ceph -s

Possible causes:

- The placement node does not exist.
- The node has not joined the ceph orch.
- Port conflicts.
- Failed to pull the container image.
- Abnormal RGW configuration.
- Abnormal node network.

Troubleshooting steps:

    ceph orch host ls
    ceph orch ps --daemon_type rgw --format json-pretty
    ss -lntp | grep 7480
    ceph health detail

---

### 22.2 Failed to Access RGW Using curl

Command:

    curl -I http://10.0.0.31:7480

If the connection fails, check:

- Whether the RGW is running.
- Whether port 7480 is being listened on.
- Whether the firewall is blocking it.
- Whether the network between the client and the RGW node is accessible.
- Whether the incorrect node or port was accessed.

On the node, check:

    ss -lntp | grep 7480

---

### 22.3 AWS CLI Authentication Failed

Common error messages:

    InvalidAccessKeyId
    SignatureDoesNotMatch
    AccessDenied

Troubleshooting steps:

    radosgw-admin user info --uid=s3-user
    cat ~/.aws/credentials
    cat ~/.aws/config

Verify:

- The AccessKey is correct.
- The SecretKey is correct.
- The correct profile is being used.
- The endpoint-url is correct.
- The region settings are consistent.
- Whether the path-style has been configured correctly.
- Whether the system time is accurate.

Out-of-sync times can also cause signature verification issues.

Check:

    timedatectl

---

### 22.4 Failed to Create a Bucket

Common causes:

- Abnormal user permissions.
- The bucket name does not follow the rules.
- The bucket already exists.
- Incorrect endpoint.
- Abnormal RGW service.
- Abnormal backend pool.

Troubleshooting steps:

    aws --profile ceph-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls
    radosgw-admin user info --uid=s3-user
   RGW utilizes multiple underlying pools to store metadata, indexes, and object data. It is not advisable to directly delete a pool simply because you wish to clean up objects. The proper procedure involves:

- Deleting objects through the S3 API.
- Managing users and buckets using radosgw-admin.
- Deleting a pool should only be considered in experimental scenarios where the entire object storage environment is deemed obsolete.

---

### 24.6 Large Numbers of Small Objects Require Special Attention

A large number of small objects in object storage can affect various aspects, including:

- Bucket indexing
- Metadata management
- RGW request load
- The number of OSD objects
- Listing performance
- Statistical analysis performance
- Backup and migration efficiency

In production environments, it is essential to conduct stress tests based on the size, quantity, and access patterns of the business objects.

---

## Chapter 25: Advanced SRE Methodologies

### 25.1 RGW Troubleshooting Should Be Conducted in Three Layers

**Layer 1: Client**
- Verify whether the endpoint is correct.
- Ensure that the AccessKey/SecretKey are valid.
- Check if the path-style or virtual-host-style matches the requirements.
- Confirm that time synchronization is in place.
- Verify if requests are passing through a proxy.

**Layer 2: RGW Gateway**
- Determine if the RGW daemon is running.
- Check whether the required ports are being listened on.
- Assess the health of multiple instances.
- Verify that Nginx/LB are functioning properly.
- Check for any 4xx/5xx errors in the logs.

**Layer 3: Ceph Backend**
- Confirm that ceph -s is running smoothly.
- Verify whether OSDs are up and available.
- Ensure that PGs are active and clean.
- Check if RGW pools are nearfull.
- Assess if there are any abnormalities with the bucket indexing.

---

### 25.2 Object Storage Is Not a File System

Object storage does not follow the semantics of traditional POSIX file systems. Although object names can be structured like:

    logs/2026/04/app.log

this is not a strict directory structure but rather a prefix for the object key. If your business requires shared directories and file system semantics, consider using CephFS. For cloud block storage needs, RBD may be a better option.

---

### 25.3 The Core of RGW in Production Environments Is Access Control

Getting RGW up and running is just the first step. In production, it is crucial to implement:

- Unified access points.
- HTTPS security.
- Multiple instances for high availability.
- Load balancing for distributing requests.
- Secure key management.
- Quota control to manage resource usage.
- Monitoring and alerts for timely issues detection.
- Capacity planning to ensure efficient operation.
- Failure drills to prepare for potential disruptions.
- Proper data lifecycle management.

---

## Chapter 26: Interview Answer Guidelines

If you are interviewed about Ceph RGW, you can answer the following:

- **What is Ceph RGW? How is it deployed and used?**
  - Ceph RGW is the object storage gateway for Ceph, providing HTTP APIs compatible with S3 and Swift. It does not act as a block device or file system but serves as an object storage service tailored for buckets and objects. It can be deployed using cephadm by specifying service ID, number of nodes, and listening ports.

- **How to use RGW?**
  - Users need to create accounts through radosgw-admin and obtain AccessKey and SecretKey. They can then use tools like AWS CLI, s3cmd, mc, or S3 SDKs to interact with RGW endpoints for bucket creation, object upload/download, and deletion.

- **What are the considerations in a production environment?**
  - In production, it is recommended to use HTTPS for secure access, place frontends like Nginx/HAProxy/LB in front of RGW, and manage AccessKey security. It is also important to monitor key metrics such as AccessKey usage, user quotas, bucket quotas, RGW pool capacity, request error rates, object counts, bucket indexing status, OSD health, and PG status.

---

## Chapter 27: Summary of This Article

This article covers the key aspects of Ceph RGW object storage:

1. RGW serves as the object storage gateway for Ceph.
2. It provides HTTP APIs compatible with S3/Swift.
3. Data is stored in Ceph RADOS and OSD clusters.
4. RGW is suitable for scenarios involving images, attachments, backups, archives, logs, etc.
5. It is not designed to be used as a block device or shared file system.
6. cephadm can be used to deploy RGW services.
7. In experiments, port 7480 is commonly used for RGW HTTP access.
8. In production, HTTPS should be used