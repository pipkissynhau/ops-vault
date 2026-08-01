# MinIO Permission Management: Users, Policy, AccessKey, and SecretKey

Recommended path: 05-Storage/02-MinIO/07-MinIO Permission Management: Users, Policy, AccessKey, and SecretKey.md

Tags: #MinIO #AuthorityManagement #UserManagement #Policy #AccessKey #SecretKey #S3 #ObjectStorage #Secure. #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the seventh article of the MinIO module, focusing on learning MinIO's permission management system.

Previously completed:

- MinIO object storage basics
- Bucket / Object / Prefix
- S3 API basics
- Single-node single-disk deployment
- Single-node multi-disk deployment
- 4-node multi-disk distributed cluster deployment
- Internal HTTP and external HTTPS entry design
- Nginx HTTPS unified entry
- mc client configuration, Bucket management, and object operations

This article enters the core of MinIO security governance.

This article focuses on solving:

    Can the root user be used for business?
    What are AccessKey / SecretKey?
    How to create MinIO users?
    What is a Policy?
    How to bind read-only permissions to a user?
    How to bind read-write permissions to a user?
    How to restrict a user to access only specific Buckets?
    How to restrict a user to access only specific Prefixes?
    What to do if the key is leaked?
    How to design minimal permissions in production?
    How to avoid accidental deletion of Buckets or objects?

This article emphasizes hands-on practice, with executable commands provided for all core operations.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the boundaries of the MinIO root user.
2. Understand the role of AccessKey and SecretKey.
3. Understand the relationship between MinIO users, Policy, Buckets, and Objects.
4. Create regular business users using mc.
5. Write a read-only Policy.
6. Write a read-write Policy.
7. Write a Policy for specific Bucket access.
8. Write a Policy for specific Prefix access.
9. Bind a Policy to a specific user.
10. Configure mc alias using a regular user.
11. Verify that a read-only user cannot upload or delete.
12. Verify that a read-write user can upload and download.
13. Verify that a user can only access authorized Buckets.
14. Master user disable, enable, and deletion methods.
15. Master the handling process after AccessKey / SecretKey leakage.
16. Establish a production MinIO permission governance baseline.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This article continues from the previous MinIO distributed cluster:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc Client |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entry |

---

### 3.2 Access Entry

Backend direct access entry:

    http://10.0.0.41:9000

HTTPS unified entry:

    https://s3.minio.local

Console entry:

    https://console.minio.local

This article defaults to using:

    https://s3.minio.local

If the experimental environment uses a self-signed certificate, temporarily use:

    --insecure

in mc commands. Do not use --insecure long-term in production.

---

### 3.3 Image Version

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

---

## IV. Understanding MinIO Permission Model

### 4.1 Core Objects of Permission Management

MinIO permission management mainly revolves around the following objects:

| Object | Description |
|---|---|
| User | User, owns AccessKey / SecretKey |
| AccessKey | Access key, similar to username |
| SecretKey | Secret key, similar to password |
| Policy | Permission policy, defines what can be done |
| Bucket | Object storage container |
| Object | Object data |
| Prefix | Object Key prefix, similar to directory |
| Group | User group, can be used for batch authorization |

This article focuses on learning:

    User
    AccessKey
    SecretKey
    Policy
    Bucket
    Prefix

---

### 4.2 What is the root user

The root user is set via environment variables when MinIO starts:

    MINIO_ROOT_USER
    MINIO_ROOT_PASSWORD

In the current experiment:

    MINIO_ROOT_USER=minioadmin
    MINIO_ROOT_PASSWORD=MinioAdmin@123456

The root user has the highest permissions.

The root user is suitable for:

    Initializing the cluster
    Creating administrator users
    Creating business users
    Creating Policies
    Binding Policies
    Troubleshooting and emergency management
    Managing Console

The root user is not suitable for:

    Long-term use by business applications
    Writing to code repositories
    Writing to CI/CD plain text variables
    Sharing with multiple businesses
    Arbitrary use by developers
    Configuring in production applications

Production principles:

    The root user is only for platform management.
    Business applications must use regular users.
    Regular users must be bound to minimal permission Policies.
    Each business should have independent AccessKey.
    Keys can be individually disabled and rotated after leakage.

---

### 4.3 What is a Policy

A Policy is a permission strategy, described in JSON to define what resources a user can access and what operations they can perform.

Policies typically include:

    Version
    Statement
    Effect
    Action
    Resource

Simple understanding:

    Effect: Allow means permission is granted.
    Action represents the allowed operations.
    Resource represents the allowed Bucket or object path.

Common Actions:

| Action | Purpose |
|---|---|
| s3:ListBucket | List objects in the Bucket |
| s3:GetObject | Download an object |
| s3:PutObject | Upload an object |
| s3:DeleteObject | Delete an object |
| s3:GetBucketLocation | Get Bucket location information |
| s3:ListAllMyBuckets | View all Buckets (use with caution) |

---

### 4.4 Bucket-level and Object-level Resources

Bucket-level resources:

    arn:aws:s3:::bucket-name

Object-level resources:

    arn:aws:s3:::bucket-name/*
    arn:aws:s3:::bucket-name/prefix/*
    arn:aws:s3:::bucket-name/uploads/*
    arn:aws:s3:::bucket-name/logs/app01/*

Differences:

    ListBucket operates on Bucket-level resources.
    GetObject / PutObject / DeleteObject operate on Object-level resources.

Common errors:

    Only writing arn:aws:s3:::bucket/*
    Without writing arn:aws:s3:::bucket

This may cause users to fail listing operations.

---

## Five. Experiment Task Planning

This document will complete the following experiments:

| Experiment | Content |
|---|---|
| Experiment 1 | Create a test Bucket |
| Experiment 2 | Create a read-only Policy |
| Experiment 3 | Create a read-only user and bind the Policy |
| Experiment 4 | Verify the read-only user can download but cannot upload |
| Experiment 5 | Create a read-write Policy |
| Experiment 6 | Create a read-write user and bind the Policy |
| Experiment 7 | Verify the read-write user can upload and download |
| Experiment 8 | Create a Prefix-level permission Policy |
| Experiment 9 | Verify the user can only access the specified Prefix |
| Experiment 10 | User disable, enable, and deletion |
| Experiment 11 | Emergency response for key leakage |
| Experiment 12 | Production permission governance check |

---

## Six. Prepare mc Command Environment

### 6.1 Create Directories

Execute on minio-client or management node:

    mkdir -p /data/minio/mc-config
    mkdir -p /tmp/minio-policy-demo/policies
    mkdir -p /tmp/minio-policy-demo/files
    mkdir -p /tmp/minio-policy-demo/download

---

### 6.2 Configure Administrator alias

If using HTTPS unified entry with self-signed certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

If using formal trusted certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

---

### 6.3 Create Simplified Function

To reduce command length, temporarily define:

    export MC_IMAGE=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
    export MC_CONFIG=/data/minio/mc-config
    export MC_WORKDIR=/tmp/minio-policy-demo

    mcx() {
      docker run --rm \
        -v ${MC_CONFIG}:/root/.mc \
        -v ${MC_WORKDIR}:/demo \
        ${MC_IMAGE} "$@"
    }

If using self-signed certificate, subsequent commands can be written as:

    mcx --insecure ls minio-admin

Note:

    This function is only valid in the current shell session.
    It needs to be redefined after reopening the terminal.
    Most commands in this document retain the full docker run format, for easy copying to notes for long-term storage.

---

## Seven. Experiment 1: Create Test Bucket

### 7.1 Create Bucket

Create a Bucket for permission testing:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mb minio-admin/policy-demo

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-admin

---

### 7.2 Upload Base Test Objects

Create test file: /think

```
echo "public read test file" > /tmp/minio-policy-demo/files/readme.txt
echo "app01 upload file" > /tmp/minio-policy-demo/files/app01.txt
echo "app02 upload file" > /tmp/minio-policy-demo/files/app02.txt

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/files/readme.txt minio-admin/policy-demo/readme.txt

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/files/app01.txt minio-admin/policy-demo/app01/app01.txt

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/files/app02.txt minio-admin/policy-demo/app02/app02.txt

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-admin/policy-demo

---

## VIII. Experiment 2: Creating a Read-Only Policy

### 8.1 Read-Only Policy Objectives

Create a read-only policy:

    Users can view the policy-demo Bucket.
    Users can download objects in the policy-demo Bucket.
    Users cannot upload objects.
    Users cannot delete objects.
    Users cannot access other Buckets.

Policy name:

    policy-demo-readonly

---

### 8.2 Writing the Read-Only Policy File

Create file:

    cat > /tmp/minio-policy-demo/policies/policy-demo-readonly.json <<'EOF'
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
            "arn:aws:s3:::policy-demo"
          ]
        },
        {
          "Effect": "Allow",
          "Action": [
            "s3:GetObject"
          ],
          "Resource": [
            "arn:aws:s3:::policy-demo/*"
          ]
        }
      ]
    }
    EOF

View:

    cat /tmp/minio-policy-demo/policies/policy-demo-readonly.json

---

### 8.3 Creating the Policy

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy create minio-admin policy-demo-readonly /demo/policies/policy-demo-readonly.json

View Policy:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy info minio-admin policy-demo-readonly

Note:

    If an older version of mc uses policy add, change it according to the actual prompt:
    mc admin policy add <alias> <policy-name> <policy-file>
    This note defaults to using policy create.

---

## IX. Experiment 3: Creating a Read-Only User and Binding the Policy

### 9.1 Creating a Read-Only User

User:

    policy-readonly-user

Password:

    ReadOnlyUser@123456

Execute: /think

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin user add minio-admin policy-readonly-user 'ReadOnlyUser@123456'

View user:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin user info minio-admin policy-readonly-user

---

### 9.2 Attach Read-Only Policy

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin policy attach minio-admin policy-demo-readonly --user policy-readonly-user

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin user info minio-admin policy-readonly-user

---

## TenI don't know.Experiment Four: Verify Read-Only User Permissions

### 10.1 Configure Read-Only User alias

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure alias set minio-readonly https://s3.minio.local policy-readonly-user 'ReadOnlyUser@123456'

View alias:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure alias list

---

### 10.2 Verify Can List Bucket Contents

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls minio-readonly/policy-demo

Expected:

You can see objects or prefixes such as readme.txt, app01/, app02/.

---

### 10.3 Verify Can Download Objects

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-policy-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp minio-readonly/policy-demo/readme.txt /demo/download/readme-readonly.txt

View:

cat /tmp/minio-policy-demo/download/readme-readonly.txt

Expected:

public read test file

---

### 10.4 Verify Cannot Upload Objects

Create file:

echo "readonly user should not upload" > /tmp/minio-policy-demo/files/readonly-upload.txt

Attempt upload:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-policy-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp /demo/files/readonly-upload.txt minio-readonly/policy-demo/readonly-upload.txt

Expected:

Upload fails.
Returns Access Denied or insufficient permissions.

---

### 10.5 Verify Cannot Delete Objects

Attempt deletion:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure rm minio-readonly/policy-demo/readme.txt

Expected:

Deletion fails.
Returns Access Denied or insufficient permissions.

Conclusion:

Read-only users can list and download.
Read-only users cannot upload.
Read-only users cannot delete.

---

## ElevenI don't know.Experiment Five: Create Read-Write Policy

### 11.1 Read-Write Policy Objective

Create a read-write policy: /think

Users can list the policy-demo Bucket.  
Users can download objects.  
Users can upload objects.  
Users can delete objects.  
Users cannot access other Buckets.

Policy Name:

    policy-demo-readwrite

---

### 11.2 Writing the Read-Write Policy File

Create file:

    cat > /tmp/minio-policy-demo/policies/policy-demo-readwrite.json <<'EOF'
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
            "arn:aws:s3:::policy-demo"
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
            "arn:aws:s3:::policy-demo/*"
          ]
        }
      ]
    }
    EOF

---

### 11.3 Creating the Policy

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy create minio-admin policy-demo-readwrite /demo/policies/policy-demo-readwrite.json

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy info minio-admin policy-demo-readwrite

---

## Twelve、Experiment Six: Creating a Read-Write User and Binding the Policy

### 12.1 Creating the Read-Write User

User:

    policy-readwrite-user

Password:

    ReadWriteUser@123456

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user add minio-admin policy-readwrite-user 'ReadWriteUser@123456'

Attach Policy:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy attach minio-admin policy-demo-readwrite --user policy-readwrite-user

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin policy-readwrite-user

---

### 12.2 Configuring the Read-Write User alias

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-readwrite https://s3.minio.local policy-readwrite-user 'ReadWriteUser@123456'

---

## Thirteen、Experiment Seven: Verifying the Read-Write User Permissions

### 13.1 Verifying the Ability to Upload Objects

Create file:

    echo "readwrite user upload test" > /tmp/minio-policy-demo/files/readwrite-upload.txt

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/files/readwrite-upload.txt minio-readwrite/policy-demo/readwrite-upload.txt

Check: /tmp/minio-policy-demo/files/readwrite-upload.txt

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls minio-readwrite/policy-demo

---

### 13.2 Verify Downloading Objects

Download:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-readwrite/policy-demo/readwrite-upload.txt /demo/download/readwrite-download.txt

Check:

    cat /tmp/minio-policy-demo/download/readwrite-download.txt

---

### 13.3 Verify Deleting Objects

Delete:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm minio-readwrite/policy-demo/readwrite-upload.txt

Confirm:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-readwrite/policy-demo

---

### 13.4 Verify Cannot Access Other Buckets

Create another Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mb minio-admin/other-demo

Attempt access with readwrite user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-readwrite/other-demo

Expected:

    Access failed.
    Because policy-demo-readwrite only authorizes policy-demo.

---

## FourteenI don't know.Experiment 8: Create Prefix-Level Policy

### 14.1 Prefix Policy Target

Create a user that only allows access to:

    policy-demo/app01/*

Deny access to:

    policy-demo/app02/*
    policy-demo/readme.txt
    Other Buckets

Policy name:

    policy-demo-app01-readwrite

User:

    app01-user

---

### 14.2 Write Prefix-Level Policy

Create file:

    cat > /tmp/minio-policy-demo/policies/policy-demo-app01-readwrite.json <<'EOF'
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:GetBucketLocation"
          ],
          "Resource": [
            "arn:aws:s3:::policy-demo"
          ]
        },
        {
          "Effect": "Allow",
          "Action": [
            "s3:ListBucket"
          ],
          "Resource": [
            "arn:aws:s3:::policy-demo"
          ],
          "Condition": {
            "StringLike": {
              "s3:prefix": [
                "app01/*"
              ]
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "s3:GetObject",
            "s3:PutObject",
            "s3:DeleteObject"
          ],
          "Resource": [
            "arn:aws:s3:::policy-demo/app01/*"
          ]
        }
      ]
    }
    EOF

Explanation:

    ListBucket applies to the Bucket.
    GetObject / PutObject / DeleteObject apply to app01/*.
    Condition restricts listing Prefix.
    This way, app01-user can only operate around the app01/ prefix.

---

### 14.3 Create Prefix-Level Policy

Execute: /think

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-policy-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin policy create minio-admin policy-demo-app01-readwrite /demo/policies/policy-demo-app01-readwrite.json

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin policy info minio-admin policy-demo-app01-readwrite

---

## 15. Experiment 9: Create app01 User and Verify Prefix Permissions

### 15.1 Create app01 User

User:

app01-user

Password:

App01User@123456

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin user add minio-admin app01-user 'App01User@123456'

Attach Policy:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin policy attach minio-admin policy-demo-app01-readwrite --user app01-user

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin user info minio-admin app01-user

---

### 15.2 Configure app01 User Alias

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure alias set minio-app01 https://s3.minio.local app01-user 'App01User@123456'

---

### 15.3 Verify Access to app01 Prefix

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls minio-app01/policy-demo/app01/

Download:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-policy-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp minio-app01/policy-demo/app01/app01.txt /demo/download/app01-download.txt

View:

cat /tmp/minio-policy-demo/download/app01-download.txt

---

### 15.4 Verify Upload to app01 Prefix

Create file:

echo "app01 new object" > /tmp/minio-policy-demo/files/app01-new.txt

Upload:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-policy-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp /demo/files/app01-new.txt minio-app01/policy-demo/app01/app01-new.txt

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls minio-app01/policy-demo/app01/

---

### 15.5 Verify Inability to Access app02 Prefix

Attempt to view app02: /think

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-app01/policy-demo/app02/

Expected:

    Access denied or insufficient permissions.

Attempt to upload to app02:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/files/app1-new.txt minio-app01/policy-demo/app02/app1-new.txt

Expected:

    Upload failed.
    Returns Access Denied or insufficient permissions.

Conclusion:

    Prefix-level permissions can isolate different business directories within the same Bucket.
    However, a clearer practice in production is typically to split by business into separate Buckets or Prefixes, and establish clear naming conventions.

---

## SixteenI don't know.User Management Operations

### 16.1 View User List

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user list minio-admin

---

### 16.2 View User Information

View read-only user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin policy-readonly-user

View app01 user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin app01-user

---

### 16.3 Disable User

Disable read-only user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user disable minio-admin policy-readonly-user

Verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-readonly/policy-demo

Expected:

    Access denied.

---

### 16.4 Enable User

Enable:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user enable minio-admin policy-readonly-user

Verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-readonly/policy-demo

---

### 16.5 Delete User

High-risk warning:

    Deleting a user will cause business using this user's AccessKey to lose access to object storage.
    Before deletion in production, confirm that business has switched to new keys.
    It is recommended to first disable the user, observe no impact, then delete.

Delete:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user remove minio-admin policy-readonly-user

Verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user list minio-admin

Note:

    If this experiment needs the policy-readonly-user again later, it can be recreated.

---

## SeventeenI don't know.Policy Management Operations

### 17.1 View Policy List

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin policy list minio-admin

---

### 17.2 View Policy Content

Run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy info minio-admin policy-demo-readwrite

---

### 17.3 Detach User Policy

Example: Detach policy from app01-user.

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy detach minio-admin policy-demo-app01-readwrite --user app01-user

Verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin app01-user

Re-attach:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy attach minio-admin policy-demo-app01-readwrite --user app01-user

---

### 17.4 Delete Policy

High-risk warning:

    Must confirm no users or business dependencies before deleting a policy.
    Users dependent on this policy may lose access after deletion.

Deletion example:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy remove minio-admin policy-demo-readonly

Notes:

    If the current version command shows "remove" is not supported, use the corresponding subcommand based on mc help information.
    Suggest running "admin policy list" and "admin user info" before deletion for verification.

---

## EighteenI don't know.AccessKey and SecretKey Governance

### 18.1 Nature of AccessKey and SecretKey

AccessKey / SecretKey are credentials for accessing object storage.

Similar risk level:

    Database account password
    Cloud platform AK/SK
    GitLab Token
    Kubernetes kubeconfig
    Harbor robot token

After leakage, may lead to:

    Data being downloaded.
    Data being modified.
    Data being deleted.
    Bucket being emptied.
    Business access anomalies.
    Cost or capacity abnormal growth.
    Sensitive file leakage.

---

### 18.2 Key Storage Principles

Do not do in production:

    Write to Git repository.
    Write to public Markdown.
    Write to screenshots.
    Send to group chats.
    Hardcode in images.
    Hardcode in frontend code.
    Multiple businesses share a key group.
    Former employees retain keys.

Should do in production:

    Use Secret management.
    Use CI/CD key variables.
    Use configuration center encrypted fields.
    Each business has independent users.
    Each environment has independent users.
    Permissions limited to specific Bucket or Prefix.
    Regular key rotation.
    Immediately disable after leakage.

---

### 18.3 Key Naming Suggestions

Recommended naming:

    app01-prod
    app01-test
    backup-prod
    log-archive-prod
    devops-artifacts-prod

Not recommended:

    test
    user1
    admin2
    aaa
    minio-user

Principles:

    Clearly identify business.
    Clearly identify environment.
    Clearly identify purpose.

---

## NineteenI don't know.Key Leakage Emergency Response

### 19.1 First Step After Key Leakage Discovery

Do not delete Bucket first.

First, confirm:

    Which AccessKey was leaked?
    Which user is associated?
    Which Policies are bound?
    Which Buckets can be accessed?
    Does it have write/delete permissions?
    Are there any abnormal accesses?
    Is immediate disablement needed?

---

### 19.2 Immediately Disable User

If key leakage is confirmed, prioritize disabling the user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user disable minio-admin app01-user

Notes:

    "disable" is safer than "remove".
    "disable" can quickly block access.
    Decide on deletion or recreation after confirming business switch.

---

### 19.3 Create New User or Key

Recommended approach: /think

Create a new user.
Attach the same or more restrictive Policy.
Modify business configurations.
Verify business normal operation.
Monitor logs.
Delete or retain disabled old users for a period of time.

Example of creating a new user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user add minio-admin app01-user-v2 'App01UserV2@123456'

Attach Policy:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy attach minio-admin policy-demo-app01-readwrite --user app01-user-v2

---

### 19.4 Troubleshooting Abnormal Access

Check directions:

    Nginx access.log
    MinIO logs
    Bucket object changes
    Presence of large downloads
    Presence of large deletions
    Presence of abnormal uploads
    Presence of unknown IP access
    Presence of abnormal capacity growth

Command examples:

    tail -f /var/log/nginx/access.log
    tail -f /var/log/nginx/error.log
    docker logs minio

View objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-admin/policy-demo

---

### 19.5 Emergency Process Summary

Key leakage handling process:

    1. Confirm the leaked AccessKey.
    2. Query the corresponding user and Policy.
    3. Immediately disable the user.
    4. Check access logs and object changes.
    5. Create a new user or new key.
    6. Attach minimal permission Policy.
    7. Modify business configurations.
    8. Verify business access normality.
    9. Observe for a period of time.
    10. Delete old users or keep them disabled.
    11. Review the cause of the leakage.
    12. Strengthen Secret management.

---

## Twenty, Production Permission Design Methods

### 20.1 Split by Business

Recommendations:

| Business | Bucket | User |
|---|---|---|
| app01 | app01-uploads-prod | app01-prod |
| app02 | app02-uploads-prod | app02-prod |
| backup | backup-prod | backup-prod |
| logs | logs-archive-prod | logs-prod |
| devops | devops-artifacts-prod | devops-prod |

Benefits:

    Clear permissions.
    Clear capacity ownership.
    Clear alert ownership.
    Small impact range of key leakage.
    Easy cleanup after business decommissioning.

---

### 20.2 Split by Environment

Production and testing should not share keys.

Recommendations:

    app01-dev
    app01-test
    app01-prod

Corresponding Buckets:

    app01-uploads-dev
    app01-uploads-test
    app01-uploads-prod

Reasons:

    Avoid test misoperations on production.
    Avoid developers holding production keys.
    Facilitate independent rotation.
    Facilitate auditing.

---

### 20.3 Split by Permissions

The same business can also split permissions:

| User | Permissions |
|---|---|
| app01-reader | Read-only |
| app01-writer | Upload and download |
| app01-admin | Management permissions, use with caution |
| app01-backup | Backup synchronization permissions |

Principles:

    Grant only the necessary permissions.
    Read services should not have delete permissions.
    Backup services should not have permissions for irrelevant Buckets.
    Temporary users should set lifecycle or expiration cleanup.

---

### 20.4 Use ListAllMyBuckets with Caution

In production, it's not recommended for ordinary business users to have:

    s3:ListAllMyBuckets

Reasons:

    They may see unrelated Bucket names.
    Increases the attack surface.
    Violates the principle of least privilege.

Ordinary business users should only access the Buckets they need.

---

## Twenty-one, Common Issue Troubleshooting

### 21.1 Access Denied

Common causes:

    User not bound to a Policy.
    Policy has incorrect Bucket name.
    Resource missing Bucket-level ARN.
    Resource missing Object-level ARN.
    Action does not include PutObject / GetObject / DeleteObject.
    Prefix Condition is incorrect.
    alias used incorrect user.
    Operated on unauthorized Bucket.

Troubleshoot:

    admin user info
    admin policy info
    mc ls
    mc stat
    Check JSON Policy

---

### 21.2 User Can Upload but Cannot List Objects

Common causes:

    Has s3:PutObject.
    Lacks s3:ListBucket.

Resolution:

    Add s3:ListBucket at the Bucket level in Resource.

---

### 21.3 User Can Download but Cannot View Bucket List

Common causes:

    No ListBucket.
    No ListAllMyBuckets.
    Only granted GetObject.

Note:

    This is not necessarily an issue.
    If the application knows the full object Key, it may only need GetObject.
    However, mc ls requires ListBucket permission.

---

### 21.4 Prefix Permissions Not Taking Effect

Common Causes:

    Prefix written as /app01/*.
    Correct should be app01/*.
    Resource written incorrectly.
    Condition written incorrectly.
    Upload target is not app01/.
    mc uses an old alias or old user.
    User bound to multiple Policies, permissionsOverlay.

Troubleshooting:

    admin user info
    admin policy info
    mc alias list
    mc ls with specified Prefix
    Test app01 and app02 paths

---

### 21.5 Policy Changes Not Reflecting Permissions

Possible Causes:

    Modified local JSON but did not re-create/update.
    User bound to another Policy.
    mc alias uses another user.
    Browser or client cache.
    Old AccessKey still valid.

Resolution:

    Recheck policy info.
    Check user info.
    Reconfigure alias.
    Re-test.

---

## Twenty-two, High-Risk Operations Reminder

The following operations must be cautious in production environments:

    Bind consoleAdmin to business users.
    Bind overly broad permissions to business users.
    Bind s3:* to business users.
    Bind Resource: * to business users.
    Business uses root user.
    Delete users.
    Delete Policies.
    Modify production Policies.
    Disable production users.
    Do not handle after AccessKey leakage.
    Multiple businesses share a group of AccessKeys.
    Write SecretKey to Git.

Confirm before execution:

    Whether current environment is production.
    Whether current user has business in use.
    Which Buckets are affected by current Policy.
    Whether there is business confirmation.
    Whether there is a rollback plan.
    Whether there is operation records.

---

## Twenty-three, Experiment Cleanup

### 23.1 Delete Test Objects and Buckets

High-Risk Reminder:

    Only clean up experimental Buckets.
    Do not accidentally delete production Buckets.

Delete policy-demo objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-admin/policy-demo

Delete Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-admin/policy-demo

Delete other-demo:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-admin/other-demo

If other-demo is not empty, clear it first before deletion.

---

### 23.2 Delete Experimental Users

Before deletion, it is recommended to first disable, then remove.

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user disable minio-admin policy-readwrite-user

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user remove minio-admin policy-readwrite-user

Delete app01-user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user remove minio-admin app01-user

Delete app01-user-v2:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user remove minio-admin app01-user-v2

If policy-readonly-user has already been deleted, ignore the error.

---

### 23.3 Delete Experimental Policies

Delete:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy remove minio-admin policy-demo-readwrite

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin policy remove minio-admin policy-demo-app01-readwrite

If policy-demo-readonly still exists:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure admin policy remove minio-admin policy-demo-readonly

---

### 23.4 Clean Up Local Files

    rm -rf /tmp/minio-policy-demo

---

## Twenty-Four, Production Permission Governance Checklist

### 24.1 User Check

| Check Item | Requirement | Result |
|---|---|---|
| Root User | Only for management |  |
| Business User | Each business has independent user |  |
| Environment Isolation | dev/test/prod do not share users |  |
| Departed Personnel | Accounts should be cleaned up promptly |  |
| Abandoned Business | Users should be disabled or deleted |  |
| User Naming | Should reflect business and environment |  |

---

### 24.2 Policy Check

| Check Item | Requirement | Result |
|---|---|---|
| Minimum Privilege | Only necessary Actions |  |
| Bucket Scope | Only authorize necessary Buckets |  |
| Prefix Scope | Limit Prefix when necessary |  |
| Delete Permission | Grant DeleteObject cautiously |  |
| Management Permission | Not for regular business users |  |
| Wildcard Permission | Use Resource: * cautiously |  |

---

### 24.3 Key Check

| Check Item | Requirement | Result |
|---|---|---|
| Git Leak | Not allowed |  |
| Plain Text Scripts | Not allowed |  |
| Group Chat Spread | Not allowed |  |
| Regular Rotation | Recommended |  |
| Leak Emergency | Has process |  |
| Secret Management | Use encrypted configuration or Secret system |  |

---

### 24.4 Operation Check

| Check Item | Requirement | Result |
|---|---|---|
| Delete Bucket | Must be approved |  |
| Delete Prefix | Must confirm path |  |
| Modify Policy | Must assess impact |  |
| Disable User | Must confirm business impact |  |
| Delete User | First disable and observe |  |
| Root Operation | Must be recorded |  |

---

## Twenty-Five, Interview Answer Approach

If asked in an interview:

    How is MinIO permission managed? Can business applications directly use the root user?

You can answer:

    MinIO's permission management mainly revolves around users, AccessKey, SecretKey, and Policy. The root user is the cluster administrator account, suitable only for initialization, user creation, Policy management, and troubleshooting. It should not be used long-term by business applications.
    In production, I would create independent users for each business or environment, such as app01-prod and app01-test, and bind them to minimal privilege Policies. Policies control which operations users can perform via Actions (e.g., ListBucket, GetObject, PutObject, DeleteObject) and which resources they can access via Resource (e.g., specific Buckets or Prefixes).
    If a business only needs to download files, grant GetObject and necessary ListBucket; if upload is required, grant PutObject; DeleteObject should be granted cautiously, only for businesses that truly need to delete objects.
    For different business directories within the same Bucket, use Prefix isolation (e.g., allow access to app01/* but not app02/*). However, a clearer approach is typically to split Buckets by business and environment.
    AccessKey and SecretKey should be managed at the key level, not committed to Git, not written into frontend code, and not shared across multiple businesses. After a key leak, disable the corresponding user first, investigate access logs and object changes, then create new users or keys, and delete old users after business migration.
    The overall principle is: root users should not be distributed, business users should be independent, Policies should have minimal privileges, keys should be rotatable, and operations should be auditable.

---

## Twenty-Six, Summary of This Article

This article completes the practical implementation of MinIO permission management:

1. MinIO root users are only suitable for management and not for long-term business use.
2. AccessKey / SecretKey are credentials for object storage access.
3. Policies define which resources users can access and which operations they can perform.
4. ListBucket acts on bucket-level resources.
5. GetObject / PutObject / DeleteObject act on object-level resources.
6. Read-only users should only be granted ListBucket and GetObject.
7. Read-write users can be granted GetObject and PutObject, but DeleteObject should be used cautiously.
8. Prefix-level permissions can restrict users to specific prefixes.
9. Business users should be split by business, environment, and permissions.
10. Regular business users should not have root or consoleAdmin permissions.
11. s3:* and Resource:* must be used cautiously.
12. Users can be disabled, enabled, or removed.
13. After a key leak, disable the user first rather than directly deleting data.
14. Keys should not be committed to Git, written into public documents, or shared among multiple users.
15. Deleting users, deleting Policies, and recursively deleting objects are all high-risk operations.
16. Production environments should establish audit lists for users, Policies, keys, and operations.
17. Future learning will continue on MinIO data protection: Erasure Coding, node failure and disk failure recovery.

---

## Twenty-Seven, Reference Documents

MinIO Identity and Access Management:

    https://min.io/docs/minio/linux/administration/identity-access-management.html

MinIO User Management:

    https://min.io/docs/minio/linux/administration/identity-access-management/minio-user-management.html

MinIO Policy Management:

    https://min.io/docs/minio/linux/administration/identity-access-management/policy-based-access-control.html

MinIO mc admin user:

    https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-user.html

MinIO mc admin policy:

    https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-policy.html

MinIO mc alias:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-alias.html

MinIO mc cp:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-cp.html

# AWS S3 Policy Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-policy-language-overview.html

# AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html