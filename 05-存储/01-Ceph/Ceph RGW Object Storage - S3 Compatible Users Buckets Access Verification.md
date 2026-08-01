# Ceph RGW Object Storage: S3 Interface, Bucket, User & Object Management Practice

Suggested path: 05-Storage/01-Ceph/11-Ceph RGW Object Storage: S3 Interface, Bucket, User & Object Management Practice.md

Tags: #Ceph #RGW #RADOSGateway #ObjectStorage #S3 #Bucket #AccessKey #SecretKey #radosgw-admin #ReverseAgent #HTTPS #SRE #DistributedStorage #AdvancedSre

---

## I. Document Overview

This is the tenth article in the Ceph advanced SRE storage module series, focusing on the theory, practical operations, and troubleshooting methods for Ceph RGW object storage.

Previously completed:

- Ceph foundation architecture
- RADOS, MON, MGR, OSD, CRUSH
- Differences between RBD, CephFS, and RGW storage types
- cephadm cluster initialization
- OSD management
- Pool and PG
- CRUSH fault domain
- RBD block storage practice
- CephFS file storage practice

This article enters the third core usage pattern of Ceph:

    RGW object storage

RGW, full name:

    RADOS Gateway

Can be understood as:

    The S3/Swift compatible object storage gateway provided by Ceph.

RGW is suitable for:

- Image storage
- Attachment storage
- Backup archiving
- Log archiving
- Private S3 object storage
- Applications accessing files via S3 SDK
- Business scenarios compatible with cloud object storage interfaces

This article covers:

- What is RGW
- Relationship between RGW and RADOS
- Differences between RGW and MinIO
- How to deploy RGW using cephadm
- How to specify RGW nodes and ports
- How to check RGW service status
- How to create S3 users
- How to obtain AccessKey and SecretKey
- How to use AWS CLI to access RGW
- How to create Buckets
- How to upload, download, and delete objects
- How to check Bucket status
- How to configure user and Bucket quotas
- How to expose a unified entry via Nginx / LB / HTTPS
- How to troubleshoot S3 access failures, permission issues, port unavailability, and Bucket anomalies
- RGW high availability and production environment considerations

---

## II. Experiment Objectives

After completing this article, you should be able to:

1. Understand the object storage model of RGW.
2. Understand the relationship between User, Bucket, and Object.
3. Deploy RGW service using cephadm.
4. Specify RGW runtime nodes, instance count, and listening ports.
5. Check RGW service status and access ports.
6. Create S3 users using radosgw-admin.
7. Obtain user AccessKey and SecretKey.
8. Configure S3 client using AWS CLI.
9. Create Buckets using RGW Endpoint.
10. Upload, download, view, and delete Objects.
11. Check Bucket statistics.
12. Set user-level quotas.
13. Set Bucket-level quotas.
14. Understand the automatic creation and role of RGW Pools.
15. Understand the high availability concept of RGW multi-instances.
16. Understand the boundary between internal HTTP and external HTTPS.
17. Expose a unified HTTPS entry for RGW using Nginx / LB.
18. Troubleshoot RGW service anomalies, S3 access failures, permission errors, signature errors, and Bucket anomalies.

---

## III. Experiment Environment

### 3.1 Ceph Cluster Nodes

This article continues using the Ceph module experiment environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (Optional) |
| 10.0.0.35 | ceph-client | S3 Client Testing (Optional) |
| 10.0.0.36 | rgw-lb | Nginx / HAProxy / Unified Entry (Optional) |

Main experiment system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

---

### 3.2 RGW Port Planning

This experiment uses:

    RGW internal access port: 7480
    RGW protocol: HTTP
    Unified entry protocol: HTTPS (Optional)
    Unified entry port: 443 (Optional)

Notes:

    In the experimental environment, clients can directly access RGW via http://10.0.0.31:7480.
    In production environments, it's not recommended to expose RGW HTTP ports to the public internet.
    In production, it's recommended to expose a unified HTTPS entry via Nginx, HAProxy, cloud load balancer, or cephadm ingress.

---

### 3.3 Internal HTTP vs External HTTPS Boundary

Experimental recommendation:

    HTTP can be used for backend RGW instances and internal client access.
    External access must go through a unified HTTPS entry.

Reasons:

    HTTP can reduce TLS overhead and simplify troubleshooting in trusted internal networks.
    HTTPS is mandatory for public or cross-untrusted network access to prevent exposure of AccessKey, signatures, and object data during transmission.
    Whether to enable HTTPS internally in production should be decided based on compliance, security level, and performance requirements.

This article defaults to:

    RGW backend: HTTP 7480
    External entry: Nginx / LB HTTPS 443

---

## IV. What is RGW

RGW is Ceph's object storage gateway.

It provides object storage capabilities to clients via HTTP API, compatible with common S3 operations.

Object storage model:

    User
      |
      v
    Bucket
      |
      v
    Object

Example:

    User: app-user
    Bucket: images
    Object: 2026/04/28/a.jpg

RGW is not a block device or file system.

It's more like:

    Private S3
    Private OSS
    Private object storage service

---

## V. RGW Architecture Understanding

### 5.1 RGW Data Path

Simplified path:

    S3 Client / SDK / AWS CLI
      |
      v
    HTTP / HTTPS Endpoint
      |
      v
    RGW Gateway
      |
      v
    RADOS Pools
      |
      v
    OSD Cluster

Diagram: /think

```
┌─────────────────────────────┐
│       Application / S3 Client       │
└──────────────┬──────────────┘
               │ HTTP / HTTPS
               v
┌─────────────────────────────┐
│             RGW               │
│       RADOS Gateway          │
└──────────────┬──────────────┘
               v
┌─────────────────────────────┐
│        RGW Related Pools        │
└──────────────┬──────────────┘
               v
┌─────────────────────────────┐
│           OSD Cluster           │
└─────────────────────────────┘

---

### 5.2 Relationship Between RGW and RADOS

RGW itself does not directly store objects on local disks.

After receiving an S3 request, RGW writes object data, bucket metadata, index information, etc., into Ceph RADOS.

Therefore:

    RGW is the entry point.
    RADOS is the foundation.
    OSD is the final storage location.

---

### 5.3 Differences Between RGW and RBD / CephFS

| Type | Access Method | Data Model | Typical Use Cases |
|---|---|---|---|
| RBD | Block Device | Block / Image | Cloud disks, database data disks, K8s RWO PVC |
| CephFS | File System | File / Directory | Shared directories, K8s RWX PVC |
| RGW | HTTP S3 API | Bucket / Object | Images, backups, archives, attachments, object storage |

Memorization tip:

    Use like a cloud hard disk: RBD
    Use like a shared directory: CephFS
    Use like OSS / S3: RGW

---

## Six. Differences Between RGW and MinIO

RGW and MinIO can both provide S3-compatible object storage capabilities, but they have different focuses.

| Comparison Item | Ceph RGW | MinIO |
|---|---|---|
| Ecosystem | Ceph ecosystem component | Independent object storage system |
| Underlying Storage | Ceph RADOS | MinIO's own Erasure Coding |
| Deployment Complexity | Depends on a complete Ceph cluster | Relatively lighter |
| Operation Complexity | Needs understanding of Ceph, Pool, PG, OSD | More focused on object storage |
| Storage Capabilities | Unified object entry in block, file, and object storage | Specialized S3 object storage |
| Suitable Scenarios | Existing Ceph cluster needing object interface | Standalone S3 object storage |
| Advanced Capabilities | Can combine with Ceph multi-site, CRUSH, RADOS | More suitable for lightweight object storage platforms |

Simple understanding:

    If you already have Ceph, use RGW to add object storage capabilities.
    If you want to quickly deploy S3 object storage, MinIO is usually simpler.
    Ceph RGW is more suitable for unified storage foundation scenarios.
    MinIO is more suitable for dedicated object storage scenarios.

---

## Seven. Pre-Operation Checks

### 7.1 Check Ceph Cluster Status

    ceph -s

Ideal status:

    health: HEALTH_OK

At least confirm:

    mon is normal
    mgr is normal
    osd up/in
    pgs active+clean

If there are OSD down, PG degraded, or nearfull issues, do not proceed with RGW deployment.

---

### 7.2 Check Host List

    ceph orch host ls

Confirm:

    ceph-node01
    ceph-node02
    ceph-node03

Are all within Ceph orch management.

---

### 7.3 Check Service Status

    ceph orch ls
    ceph orch ps

Confirm:

    mon, mgr, osd are normal.
    No abnormal daemons.

---

### 7.4 Check Port Usage

On the node preparing for RGW deployment:

    ss -lntp | grep 7480

If the port is occupied, change the port or stop the conflicting service.

---

### 7.5 Check Client Tools

Install AWS CLI on ceph-client.

Ubuntu:

    apt update
    apt install -y awscli jq curl

Rocky Linux 9:

    dnf install -y awscli jq curl

Verify:

    aws --version
    jq --version
    curl --version

---

## Eight. Experiment Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | Deploy Single-Instance RGW | Medium |
| Experiment 2 | Deploy Multi-Instance RGW | Medium |
| Experiment 3 | Check RGW Service Status | Low |
| Experiment 4 | Create S3 User | Medium |
| Experiment 5 | Configure AWS CLI to Access RGW | Medium |
| Experiment 6 | Create Bucket and Upload Object | Low |
| Experiment 7 | Download and Delete Object | Low |
| Experiment 8 | Check Bucket and User Information | Low |
| Experiment 9 | Set User Quota and Bucket Quota | Medium |
| Experiment 10 | Expose HTTPS Unified Entry via Nginx | Medium-High |
| Experiment 11 | Clean Up Test Resources | High |
| Experiment 12 | Common Troubleshooting | Medium |

High-Risk Warning:

    Deleting a Bucket will delete object data.
    Deleting a user with --purge-data will clean the user's data.
    Deleting an RGW Pool will cause object storage data loss.
    Do not arbitrarily delete RGW-related Pools in production environments.

---

## Nine. Experiment 1: Deploy Single-Instance RGW

### 9.1 Add RGW Label

Add the rgw label to ceph-node01:

    ceph orch host label add ceph-node01 rgw

Check:

    ceph orch host ls

---

### 9.2 Deploy Single-Instance RGW

Deploy a RGW service with service ID:

    rgw-demo

Command: /think
```

```
ceph orch apply rgw rgw-demo --placement="1 ceph-node01" --port=7480

Explanation:

    rgw is the service type.
    rgw-demo is the service ID.
    --placement specifies the number of deployments and nodes.
    --port=7480 specifies the RGW listening port.

---

### 9.3 Check Deployment Status

    ceph orch ls --service_type rgw
    ceph orch ps --daemon_type rgw

Expected output:

    rgw.rgw-demo    ?:7480    1/1    running

If it's still in starting, wait:

    watch -n 3 'ceph orch ps --daemon_type rgw'

---

### 9.4 Test Port

Execute on ceph-client or any node that can access ceph-node01:

    curl -I http://10.0.0.31:7480

May see:

    HTTP/1.1 200 OK

Or:

    HTTP/1.1 403 Forbidden

Explanation:

    Returning an HTTP response indicates the RGW service port is accessible.
    403 is not necessarily an error, it may be due to missing S3 authentication information.

---

## TenI don't know.Experiment Two: Deploy Multiple RGW Instances

### 10.1 Why Need Multiple Instances

It's not recommended to have a single RGW instance in production environments.

Value of multiple instances:

- Avoid single point of failure
- Distribute requests
- Support load balancing
- Support rolling maintenance
- Improve entry availability

---

### 10.2 Add rgw Labels to Multiple Nodes

    ceph orch host label add ceph-node01 rgw
    ceph orch host label add ceph-node02 rgw
    ceph orch host label add ceph-node03 rgw

Check:

    ceph orch host ls

---

### 10.3 Deploy 3 RGW Instances

If a single instance has already been deployed, you can re-apply:

    ceph orch apply rgw rgw-demo --placement="3 ceph-node01 ceph-node02 ceph-node03" --port=7480

Check:

    ceph orch ps --daemon_type rgw

Expected:

    ceph-node01 has rgw
    ceph-node02 has rgw
    ceph-node03 has rgw

Each node listens on port 7480.

---

### 10.4 Deploy Using Label Method

You can also use the label method:

    ceph orch apply rgw rgw-demo '--placement=label:rgw count-per-host:1' --port=7480

Explanation:

    label:rgw indicates deployment to nodes with the rgw label.
    count-per-host:1 indicates one RGW instance per node.

---

## ElevenI don't know.Experiment Three: Check RGW Service Status

### 11.1 Check RGW Service

    ceph orch ls --service_type rgw

Check details:

    ceph orch ps --daemon_type rgw

---

### 11.2 Check Node Where RGW Processes Are Located

    ceph orch ps --daemon_type rgw --format json-pretty

Or:

    ceph orch ps | grep rgw

---

### 11.3 Check RGW Related Pools

After RGW starts and is used, it will create or use multiple RGW related pools.

Check:

    ceph osd pool ls | grep rgw

May see similar:

    .rgw.root
    default.rgw.log
    default.rgw.control
    default.rgw.meta
    default.rgw.buckets.index
    default.rgw.buckets.data

Explanation:

    Pool names may vary slightly depending on versions and configurations.
    Do not delete RGW related pools arbitrarily.

---

### 11.4 Check Cluster Status

    ceph -s

Confirm:

    health is normal
    RGW service is running
    pgs are active+clean

---

## TwelveI don't know.Experiment Four: Create S3 User

### 12.1 Create User

Execute on Ceph management node:

    radosgw-admin user create \
      --uid="s3-user" \
      --display-name="S3 Test User"

The output will include:

    access_key
    secret_key

Recommended to save the output:

    radosgw-admin user create \
      --uid="s3-user" \
      --display-name="S3 Test User" \
      > /root/s3-user.json

Check:

    cat /root/s3-user.json | jq

---

### 12.2 Check User Information

    radosgw-admin user info --uid="s3-user"

Key fields:

- user_id
- display_name
- keys
- access_key
- secret_key
- caps
- user_quota
- bucket_quota

---

### 12.3 Extract AccessKey and SecretKey

Use jq to extract:

    ACCESS_KEY=$(jq -r '.keys[0].access_key' /root/s3-user.json)
    SECRET_KEY=$(jq -r '.keys[0].secret_key' /root/s3-user.json)

Check:

    echo $ACCESS_KEY
    echo $SECRET_KEY

Security reminder:

    AccessKey and SecretKey are equivalent to access credentials.
    Do not commit to Git.
    Do not write to public documentation.
    Do not post to chat groups.
    Production environments should use a key management system.

---

## ThirteenI don't know.Experiment Five: Configure AWS CLI to Access RGW

### 13.1 Configure profile on ceph-client

Execute on ceph-client: /think
```

aws configure --profile ceph-rgw

Enter the following when prompted:

    AWS Access Key ID [None]: AccessKey generated in the previous step
    AWS Secret Access Key [None]: SecretKey generated in the previous step
    Default region name [None]: us-east-1
    Default output format [None]: json

---

### 13.2 Configure path-style access

RGW experiments use IP + port access, it is recommended to configure path-style:

    aws configure set profile.ceph-rgw.s3.addressing_style path

Check the configuration:

    cat ~/.aws/config
    cat ~/.aws/credentials

Explanation:

    path-style format is similar to:
        http://10.0.0.31:7480/bucket/object

    virtual-host-style format is similar to:
        http://bucket.example.com/object

When using IP addresses in the experimental environment, path-style is more direct.

---

### 13.3 Set Endpoint variable

    export RGW_ENDPOINT="http://10.0.0.31:7480"

If testing other RGW nodes:

    export RGW_ENDPOINT="http://10.0.0.32:7480"

If using a unified entry point:

    export RGW_ENDPOINT="https://s3.example.com"

---

### 13.4 Test listing Bucket

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

If a new user has no Bucket, there may be no output.

As long as there are no authentication errors, it indicates basic functionality is working.

---

## FourteenI don't know.Experiment Six: Create Bucket and Upload Object

### 14.1 Create Bucket

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 mb s3://demo-bucket

Expected output:

    make_bucket: demo-bucket

---

### 14.2 View Bucket

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

Expected output:

    demo-bucket

---

### 14.3 Create test file

    echo "hello ceph rgw s3" > /tmp/rgw-test.txt

Check:

    cat /tmp/rgw-test.txt

---

### 14.4 Upload object

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-test.txt s3://demo-bucket/rgw-test.txt

Expected output:

    upload: ../../tmp/rgw-test.txt to s3://demo-bucket/rgw-test.txt

---

### 14.5 View object

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://demo-bucket/

Expected output:

    rgw-test.txt

---

## FifteenI don't know.Experiment Seven: Download, Delete Object

### 15.1 Download object

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp s3://demo-bucket/rgw-test.txt /tmp/rgw-test-download.txt

Check:

    cat /tmp/rgw-test-download.txt

Expected output:

    hello ceph rgw s3

---

### 15.2 Delete object

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rm s3://demo-bucket/rgw-test.txt

Check:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://demo-bucket/

Expected output:

    No longer displays rgw-test.txt

---

### 15.3 Upload directory test

Create directory:

    mkdir -p /tmp/rgw-dir
    echo "file1" > /tmp/rgw-dir/file1.txt
    echo "file2" > /tmp/rgw-dir/file2.txt

Upload directory:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-dir s3://demo-bucket/dir/ --recursive

Check:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://demo-bucket/dir/

Expected output:

    file1.txt
    file2.txt

---

## SixteenI don't know.Experiment Eight: View Bucket and User Information

### 16.1 View Bucket list

    radosgw-admin bucket list

---

### 16.2 View Bucket statistics

    radosgw-admin bucket stats --bucket=demo-bucket

Can be combined with jq:

    radosgw-admin bucket stats --bucket=demo-bucket | jq

Key fields:

- bucket
- owner
- usage
- num_objects
- size
- size_actual
- placement_rule

---

### 16.3 View user information

    radosgw-admin user info --uid=s3-user | jq

---

### 16.4 View Buckets owned by user

radosgw-admin bucket list --uid=s3-user

---

### 16.5 Checking RGW-related Pool Capacity

    ceph df

Check Pool:

    ceph osd pool ls | grep rgw

Notes:

    Bucket and object data will eventually be stored in RGW-related Pools.
    In production, these Pools' capacity and PG status should be monitored.

---

## SeventeenI don't know.Experiment Nine: Setting User Quotas and Bucket Quotas

### 17.1 User-level Quotas

Set maximum capacity of 10G and maximum object count of 10000:

    radosgw-admin quota set \
      --quota-scope=user \
      --uid=s3-user \
      --max-size=10G \
      --max-objects=10000

Enable user quotas:

    radosgw-admin quota enable \
      --quota-scope=user \
      --uid=s3-user

Check user information:

    radosgw-admin user info --uid=s3-user | jq '.user_quota'

---

### 17.2 Bucket-level Quotas

Set the quota for this user's Bucket:

    radosgw-admin quota set \
      --quota-scope=bucket \
      --uid=s3-user \
      --max-size=5G \
      --max-objects=5000

Enable:

    radosgw-admin quota enable \
      --quota-scope=bucket \
      --uid=s3-user

Check:

    radosgw-admin user info --uid=s3-user | jq '.bucket_quota'

---

### 17.3 Disabling Quotas

Disable user quotas:

    radosgw-admin quota disable \
      --quota-scope=user \
      --uid=s3-user

Disable Bucket quotas:

    radosgw-admin quota disable \
      --quota-scope=bucket \
      --uid=s3-user

---

### 17.4 Quota Use Cases

Quotas are suitable for:

- Preventing single users from filling up object storage
- Controlling resource usage for test users
- Setting limits for business Buckets
- Basic governance for multi-tenant object storage
- Avoiding infinite writes from abnormal programs

Production recommendations:

    Both users and Buckets should have reasonable quotas.
    Quotas should be planned with business growth.
    Quota alerts should be integrated with monitoring systems.
    Do not wait until the cluster is near full before governance.

---

## EighteenI don't know.Experiment Ten: Exposing HTTPS Unified Entry via Nginx

### 18.1 Why a Unified Entry is Needed

If clients directly access multiple RGW nodes:

    http://10.0.0.31:7480
    http://10.0.0.32:7480
    http://10.0.0.33:7480

Issues:

- Client configuration is complex
- Node failures require switching
- No unified domain
- Difficult to enable HTTPS
- Difficult to access logs and rate limiting
- Difficult to implement canary releases and maintenance

Production recommendation:

    Client
      |
      v
    HTTPS domain / Unified entry
      |
      v
    Nginx / HAProxy / LB
      |
      v
    Multiple RGW backend instances

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

Notes:

    Expose HTTPS externally.
    Internal RGW backend uses HTTP.
    Internal network must be trusted.
    TLS should be enabled internally if across untrusted networks.

---

### 18.3 Nginx upstream Example

Install Nginx on the rgw-lb node:

Ubuntu:

    apt update
    apt install -y nginx

Rocky Linux 9:

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

```nginx
location / {
    proxy_pass http://ceph_rgw_backend;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
}
```
EOF

Check configuration:

    nginx -t

Start:

    systemctl enable --now nginx
    systemctl reload nginx

---

### 18.4 Test with HTTPS Endpoint

Client configuration hosts or DNS:

    10.0.0.36 rgw.example.com

Set Endpoint:

    export RGW_ENDPOINT="https://rgw.example.com"

Test:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

If using self-signed certificate, testing may require:

    --no-verify-ssl

But --no-verify-ssl is not recommended for production.

Production should use trusted CA certificate.

---

### 18.5 Production Recommendations

Production environment recommendations:

- Use formal domain name
- Use trusted CA certificate
- Expose only 443 externally
- Do not expose internal RGW port to public
- Configure access log
- Configure connection count and request body size
- Configure health check
- Configure rate limiting or WAF as required by business
- Do not transmit AccessKey / SecretKey through plaintext
- Object storage entry should be monitored and alarmed

---

## NineteenI don't know.RGW High Availability Design

### 19.1 Minimum Experiment

Experiment can use:

    1 RGW instance

Example:

    ceph-node01:7480

Drawbacks:

    Single point of failure

---

### 19.2 Recommended Experiment

Recommended at least:

    3 RGW instances

Example:

    ceph-node01:7480
    ceph-node02:7480
    ceph-node03:7480

Combined with:

    Nginx / HAProxy / Load Balancer

---

### 19.3 Production Recommendations

Production environment recommendations:

- At least 2-3 RGW instances
- Distribute RGW across different nodes
- Place unified entry in front
- Unified entry supports health check
- Use HTTPS externally
- RGW backend port only allows access from entry layer
- Monitor RGW service status, request volume, error rate, latency
- Monitor RGW Pool capacity and PG status
- Do not let RGW and critical OSD compete for resources excessively

---

## TwentyI don't know.Experiment Eleven: Clean Test Resources

### 20.1 Delete Objects in Bucket

Delete all objects in demo-bucket:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rm s3://demo-bucket --recursive

Check:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://demo-bucket

---

### 20.2 Delete Bucket

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rb s3://demo-bucket

Check:

    aws --profile ceph-rgw \
      --endpoint
```

```
radosgw-admin user create --uid="s3-user" --display-name="S3 Test User"
radosgw-admin user info --uid="s3-user"
radosgw-admin user list
radosgw-admin user rm --uid="s3-user"

---

### 21.3 Bucket

    radosgw-admin bucket list
    radosgw-admin bucket list --uid=s3-user
    radosgw-admin bucket stats --bucket=demo-bucket

---

### 21.4 Quota

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

## Twenty-two, Common Issues and Troubleshooting

### 22.1 RGW Service Not Started

Check:

    ceph orch ls --service_type rgw
    ceph orch ps --daemon_type rgw
    ceph -s

Possible causes:

- Placement node does not exist
- Node not added to ceph orch
- Port conflict
- Container image pull failure
- RGW configuration anomaly
- Node network anomaly

Troubleshoot:

    ceph orch host ls
    ceph orch ps --daemon_type rgw --format json-pretty
    ss -lntp | grep 7480
    ceph health detail

---

### 22.2 curl Access to RGW Failed

Command:

    curl -I http://10.0.0.31:7480

If connection fails, troubleshoot:

- Is RGW running?
- Is 7480 listening?
- Is firewall blocking?
- Is network between client and RGW node reachable?
- Did you access wrong node or port?

Check on node:

    ss -lntp | grep 7480

---

### 22.3 AWS CLI Authentication Failed

Common errors:

    InvalidAccessKeyId
    SignatureDoesNotMatch
    AccessDenied

Troubleshoot:

    radosgw-admin user info --uid=s3-user
    cat ~/.aws/credentials
    cat ~/.aws/config

Confirm:

- AccessKey is correct
- SecretKey is correct
- Correct profile is used
- endpoint-url is correct
- region setting is consistent
- path-style is configured
- System time is accurate

Time synchronization issues may also cause signature verification problems.

Check:

    timedatectl

---

### 22.4 Bucket Creation Failed

Common causes:

- User permission anomaly
- Bucket name does not conform to rules
- Bucket already exists
- Endpoint error
- RGW service anomaly
- Backend Pool anomaly

Troubleshoot:

    aws --profile ceph-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls
    radosgw-admin user info --uid=s3-user
    ceph -s
    ceph health detail
    ceph osd pool ls | grep rgw

---

### 22.5 Object Upload Failed

Common causes:

- Bucket does not exist
- User lacks permissions
- Quota is full
- RGW backend anomaly
- OSD nearfull/full
- Network interruption
- Reverse proxy limits body size

Troubleshoot:

    radosgw-admin user info --uid=s3-user | jq '.user_quota'
    radosgw-admin bucket stats --bucket=demo-bucket | jq
    ceph -s
    ceph df
    ceph osd df

If using Nginx, check:

    client_max_body_size
    proxy_request_buffering
    Nginx error log

---

### 22.6 Bucket stats Inaccurate

Statistics may sometimes have delays.

Try: /think

radosgw-admin bucket stats --bucket=demo-bucket

Check usage when necessary:

    radosgw-admin user stats --uid=s3-user

Notes:

    Object storage statistics may not provide real-time consistent views.
    Production billing, quota, and capacity reports require integration with official mechanisms and monitoring systems.

---

### 22.7 Reverse Proxy After SignatureDoesNotMatch

Common causes:

- Host header is incorrectly rewritten
- HTTP/HTTPS scheme inconsistency
- Client uses virtual-host-style but DNS does not match
- Proxy layer modifies request path
- Proxy layer processes chunked or body abnormally

Recommendations:

- Preserve Host header
- Test with path-style
- Confirm proxy_pass does not rewrite path
- Confirm HTTPS entry domain matches client endpoint
- Confirm client time synchronization

Nginx key configuration:

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_request_buffering off;

---

## Twenty-Three, RGW Monitoring Focus

At least monitor in production:

- RGW service status
- RGW instance count
- RGW request volume
- 4xx error rate
- 5xx error rate
- Request latency
- Upload/download throughput
- Bucket count
- Object count
- User quota usage
- Bucket quota usage
- RGW Pool capacity
- RGW Pool PG status
- OSD nearfull/full
- Nginx/LB backend health status
- HTTPS certificate expiration time

---

## Twenty-Four, Production Environment Considerations

### 24.1 Do not expose RGW HTTP directly to public

For experiments can use:

    http://10.0.0.31:7480

For production external use should use:

    https://s3.example.com

And provide unified entry through:

- Nginx
- HAProxy
- Cloud load balancer
- cephadm ingress
- API Gateway

---

### 24.2 AccessKey and SecretKey must be protected

Production requirements:

- Do not write to code repository
- Do not write to public documentation
- Do not send plaintext via chat tools
- Regular rotation
- Disable immediately after leakage
- Different users for different business
- Minimum permissions
- Quota limits

---

### 24.3 Users and Buckets require governance

Should plan:

- User naming convention
- Bucket naming convention
- Quota
- Lifecycle
- Access policy
- Audit logs
- Monitoring alerts
- Deletion process
- Backup or cross-cluster replication strategy

---

### 24.4 RGW is not an independent backup system

Object storage can store backup files, but RGW itself is not the complete backup strategy.

Production still requires:

- Cross-cluster replication
- Offline backup
- Version control (as needed)
- Lifecycle management
- Regular recovery verification
- Anti-accidental deletion policy

---

### 24.5 Do not delete RGW Pool arbitrarily

RGW uses multiple underlying Pools to store metadata, indexes, and object data.

Do not directly delete Pools just to clean objects.

Correct approach:

    Delete objects via S3 API.
    Manage users and Buckets via radosgw-admin.
    Pool deletion should only occur in confirmed scenarios of abandoning the entire object storage environment.

---

### 24.6 Large numbers of small objects require careful evaluation

Massive small objects in object storage affect:

- Bucket index
- Metadata management
- RGW request pressure
- OSD object count
- Listing performance
- Statistics performance
- Backup and migration efficiency

Production should pre-perform stress testing based on object size, quantity, and access patterns.

---

## Twenty-Five, Advanced SRE Methodology

### 25.1 RGW troubleshooting should be divided into three layers

First layer: Client

    Is endpoint correct?
    Are AccessKey/SecretKey correct?
    Does path-style/virtual-host-style match?
    Is time synchronized?
    Is request passed through proxy?

Second layer: RGW gateway

    Is RGW daemon running?
    Is port listening?
    Are multiple instances healthy?
    Is Nginx/LB normal?
    Are there 4xx/5xx logs?

Third layer: Ceph backend

    Is ceph -s healthy?
    Are OSDs up/in?
    Are PGs active+clean?
    Is RGW Pool nearfull?
    Is Bucket index abnormal?

---

### 25.2 Object storage is not a file system

Object storage lacks traditional POSIX file system semantics.

Although object names can be written as:

    logs/2026/04/app.log

This is not a strict directory structure but rather a prefix of the object key.

If business requires shared directory and file system semantics, consider:

    CephFS

If business requires cloud disk, consider:

    RBD

---

### 25.3 RGW's production core is entry governance

RGW running is just the first step.

Production truly needs to do:

- Unified entry
- HTTPS
- Multiple instances
- Load balancing
- Key governance
- Quota governance
- Monitoring alerts
- Capacity planning
- Failure drills
- Data lifecycle governance

---

## Twenty-Six, Interview Answer Approach

If asked in an interview:

    What is Ceph RGW? How to deploy and use it?

You can answer:

# Ceph RGW is Ceph's object storage gateway, full name RADOS Gateway, providing HTTP APIs compatible with S3 and Swift. It is not a block device or file system, but an object storage service oriented toward Buckets and Objects.  
RGW itself is merely an access entry point; actual data and metadata are written to Ceph RADOS and ultimately stored in the OSD cluster. It is suitable for scenarios such as images, attachments, backup archives, log archives, and private S3.  
Deployment can use cephadm, specifying service ID, deployment node count, and listening port via `ceph orch apply rgw`. For example, deploy multiple RGW instances on ceph-node01, ceph-node02, ceph-node03, and listen on port 7480.  
Usage requires creating users via `radosgw-admin` to obtain AccessKey and SecretKey, then using AWS CLI, s3cmd, mc, or an application's S3 SDK pointing to the RGW endpoint to complete Bucket creation, object upload/download, and deletion.  
In production environments, it is not recommended to expose RGW HTTP ports directly to the public; instead, use Nginx, HAProxy, LB, or cephadm ingress to provide a unified HTTPS entry. Also pay attention to AccessKey protection, user quotas, Bucket quotas, RGW Pool capacity, request error rates, object count, Bucket index, OSD status, and PG status.  

---

## 27. Summary of This Article  

This article mainly organizes the core content of Ceph RGW object storage:  

1. RGW is Ceph's object storage gateway.  
2. RGW provides HTTP APIs compatible with S3/Swift.  
3. RGW data is ultimately written to Ceph RADOS and OSD.  
4. RGW is suitable for object storage scenarios such as images, attachments, backups, archives, and logs.  
5. RGW is not suitable for use as a block device or shared file system.  
6. cephadm can deploy RGW services via `ceph orch apply rgw`.  
7. In experiments, port 7480 is used as the RGW HTTP port.  
8. In production, it is recommended to access RGW via a unified HTTPS entry.  
9. `radosgw-admin` can be used to create users, view users, view Buckets, and set quotas.  
10. AWS CLI can access RGW via the endpoint-url.  
11. When testing with IP + port, it is recommended to configure path-style.  
12. RGW high availability requires multiple RGW instances and frontend load balancing.  
13. AccessKey and SecretKey must be securely managed.  
14. RGW Pools should not be deleted arbitrarily.  
15. Advanced SREs troubleshooting RGW issues need to simultaneously check the client, gateway layer, and Ceph backend.  

---

## 28. Reference Documents  

Ceph Object Gateway official documentation:  

    https://docs.ceph.com/en/reef/radosgw/  

Ceph RGW cephadm service documentation:  

    https://docs.ceph.com/en/reef/cephadm/services/rgw/  

`radosgw-admin` command documentation:  

    https://docs.ceph.com/en/reef/man/8/radosgw-admin/  

RGW management documentation:  

    https://docs.ceph.com/en/latest/radosgw/admin/  

RGW HTTP Frontends:  

    https://docs.ceph.com/en/latest/radosgw/frontends/  

Ceph RGW Multi-Site:  

    https://docs.ceph.com/en/latest/radosgw/multisite/  

AWS CLI S3 command reference:  

    https://docs.aws.amazon.com/cli/latest/reference/s3/