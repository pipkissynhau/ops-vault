# MinIO Permission Management: Users, Policies, AccessKeys, and SecretKeys

Recommended path: 05-Storage/02-MinIO/07-MinIO Permission Management: Users, Policies, AccessKeys, and SecretKeys.md

Tags: #MinIO #PermissionManagement #UserManagement #Policy #AccessKey #SecretKey #S3 #ObjectStorage #SecurityEnhancement #AdvancedSRE #ProductionOps

---

## I. Document Overview

This article is the seventh in the MinIO series, focusing on understanding MinIO's permission management system.

Previously covered topics include:

- Basics of MinIO Object Storage
- Bucket / Object / Prefix
- S3 API Basics
- Single-Disk Deployment on a Single Machine
- Multi-Disk Deployment on a Single Node
- 4-Node Distributed Cluster Deployment with Multiple Disks
- Internal HTTP and External HTTPS Access Design
- Nginx as a Unified HTTPS Gateway
- mc Client Configuration, Bucket Management, and Object Operations

This article delves into the core aspects of MinIO security governance.

Key issues addressed include:

- Can the root user be used for business purposes?
- What are AccessKeys and SecretKeys?
- How to create MinIO users?
- What is a Policy?
- How to assign read-only permissions to users?
- How to assign read-write permissions to users?
- How to restrict users from accessing specific Buckets?
- How to restrict users from accessing specific Prefixes?
- What to do in case of key leakage?
- How to design minimal permissions for production environments?
- How to prevent accidental deletion of Buckets or objects?

This article emphasizes practical operations, providing executable commands for all core tasks.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the limitations of the MinIO root user.
2. Comprehend the functions of AccessKeys and SecretKeys.
3. Grasp the relationships between MinIO users, Policies, Buckets, and Objects.
4. Use mc to create regular business users.
5. Write read-only Policies.
6. Write read-write Policies.
7. Create Bucket-specific permission Policies.
8. Create Prefix-specific permission Policies.
9. Bind Policies to specific users.
10. Configure an alias for the mc client using a regular user.
11. Verify that read-only users cannot upload or delete objects.
12. Verify that read-write users can upload and download objects.
13. Ensure that users can only access authorized Buckets.
14. Learn how to disable, enable, and delete users.
15. Understand the process for handling AccessKey/SecretKey leaks.
16. Establish a baseline for MinIO permission management in production environments.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This experiment continues from the previous MinIO distributed cluster setup:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc Client |
| 10.0.0.46 | minio-entry | Nginx HTTPS Gateway |

---

### 3.2 Access Points

Direct backend access:

    http://10.0.0.41:9000

Unified HTTPS gateway:

    https://s3.minio.local

Console access:

    https://console.minio.local

This article uses the default address:

    https://s3.minio.local

If a self-signed certificate is used in the experimental environment, use the --insecure option temporarily in mc commands:

    --insecure

However, this should not be used for production environments.

---

### 3.3 Image Versions

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

---

## IV. Understanding the MinIO Permission Model

### 4.1 Core Objects of Permission Management

MinIO permission management revolves around the following objects:

| Object | Description |
|---|---|
| User | A user who possesses an AccessKey/SecretKey |
| AccessKey | An access key, similar to a username |
| SecretKey | A secret key--insecure alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

If using a formal and trusted certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

To check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

---

### 6.3 Creating a Convenient Function

To reduce the length of commands, you can define temporary variables:

    export MC_IMAGE=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
    export MC_CONFIG=/data/minio/mc-config
    export MC WORKDIR=/tmp/minio-policy-demo

    mcx() {
      docker run --rm \
        -v ${MC_CONFIG}:/root/.mc \
        -v ${MC_WORKDIR}:/demo \
        ${MC_IMAGE} "$@"
    }

If using a self-signed certificate, the subsequent command can be written as:

    mcx --insecure ls minio-admin

Note:

    This function is only valid within the current shell session. You will need to redefine it after closing and reopening the terminal. Most of the commands in this document still use the full docker run format for ease of long-term preservation in notes.

---

## Experiment 1: Creating an Experimental Bucket

### 7.1 Creating a Bucket

Create a bucket for permission testing:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mb minio-admin/policy-demo

To check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-admin

---

### 7.2 Uploading Basic Test Objects

Create test files:

    echo "public read test file" > /tmp/minio-policy-demo/files/readme.txt
    echo "app01 upload file" > /tmp/minio-policy-demo/files/app01.txt
    echo "app02 upload file" > /tmp/minio-policy-demo/files/app02.txt

Upload the files:

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

To check the files:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-1## Section 9: Experiment 3: Creating a Read-Only User and Binding It to a Policy

### 9.1 Creating a Read-Only User

User:

    policy-readonly-user

Password:

    ReadOnlyUser@123456

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user add minio-admin policy-readonly-user 'ReadOnlyUser@123456'

Viewing the User:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin policy-readonly-user

---

### 9.2 Binding the Read-Only Policy

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy attach minio-admin policy-demo-readonly --user policy-readonly-user

Viewing the Binding:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin policy-readonly-user

---

## Section 10: Experiment 4: Verifying Read-Only User Permissions

### 10.1 Configuring a Read-Only User Alias

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-readonly https://s3.minio.local policy-readonly-user 'ReadOnlyUser@123456'

Viewing the Alias:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

---

### 10.2 Verifying the Ability to List Bucket Contents

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-readonly/policy-demo

Expected Result:

    Objects or Prefixes such as readme.txt, app01/, app02/ should be visible.

---

### 10.3 Verifying the Ability to Download Objects

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-readonly/policy-demo/readme.txt /tmp/minio-policy-demo/download/readme-readonly.txt

Viewing the Result:

    The content "public read test file" should be displayed.

---

### 10.4 Verifying the Inability to Upload Objects

Creation of a File:

    echo "readonly user should not upload" > /tmp/minio-policy-demo/files/readonly-upload.txt

Attempt at Uploading:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/files/readonly-upload.txt minio-readonly/policy-demo/readonly-upload.txt

Expected Result:

    The upload should fail.
    An "Access      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy info minio-admin policy-demo-readwrite

---

## Section Twelve, Experiment Six: Creating Read/Write Users and Binding Policies

### 12.1 Creating Read/Write Users

User:

    policy-readwrite-user

Password:

    ReadWriteUser@123456

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user add minio-admin policy-readwrite-user 'ReadWriteUser@123456'

Binding Policy:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy attach minio-admin policy-demo-readwrite --user policy-readwrite-user

Verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin policy-readwrite-user

---

### 12.2 Configuring Read/Write User Aliases

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-readwrite https://s3.minio.local policy-readwrite-user 'ReadWriteUser@123456'

---

## Section Thirteen, Experiment Seven: Verifying Read/Write User Permissions

### 13.1 Verifying Object Upload Capability

Creation:

    echo "readwrite user upload test" > /tmp/minio-policy-demo/files/readwrite-upload.txt

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/files/readwrite-upload.txt minio-readwrite/policy-demo/readwrite-upload.txt

Verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-readwrite/policy-demo

---

### 13.2 Verifying Object Download Capability

Download:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-readwrite/policy-demo/readwrite-upload.txt /demo/download/readwrite-download.txt

Verification:

    cat /tmp/minio-policy-demo/download/readwrite-download.txt

---

### 13.3 Verifying Object Deletion Capability

Deletion:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm minio-readwrite/policy-demo/readwrite-upload.txt

Confirmation:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-readwrite/policy-demo

---

### 13.4 Verifying Inability to Access Other Buckets

Creation of          ],
          "Resource": [
            "arn:aws:s3:::policy-demo/app01/*"
          ]
        }
      ]
    }
    EOF

Explanation:

    ListBucket operates on the Bucket.
    GetObject, PutObject, and DeleteObject operate on app01/.
    The Condition is used to restrict the listed Prefixes.
    This way, the app01-user can only perform operations on prefixes starting with app01/.

---

### 14.3 Creating a Prefix-Level Policy

Execution:

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

## Section Fifteen: Experiment Nine: Creating the app01 User and Verifying Prefix Permissions

### 15.1 Creating the app01 User

User:

    app01-user

Password:

    App01User@123456

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user add minio-admin app01-user 'App01User@123456'

Binding Policy:

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

### 15.2 Configuring the app01 User Alias

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-app01 https://s3.minio.local app01-user 'App01User@123456'

---

### 15.3 Verifying Access to the app01 Prefix

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

### 15.4 Verifying Upload to the app01 Prefix

Creation of file:

    echo "app01 new object" > /tmp/minio-policy-demo/files/app01-new.txt

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-policy-demo:/demo```markdown
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure admin user info minio-admin app01-user

---

### 16.3 Disabling a User

Disabling a read-only user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user disable minio-admin policy-readonly-user

Verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-readonly/policy-demo

Expected result:

    Access should fail.

---

### 16.4 Enabling a User

Enabling a user:

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

### 16.5 Deleting a User

High-risk warning:

    Deleting a user will prevent businesses using that user's AccessKey from accessing object storage.
    Make sure all services have switched to new keys before deletion.
    It is recommended to first disable the user and check for any effects before deleting.

Deletion:

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

    If you need to use policy-readonly-user again in future experiments, you can create it anew.

---

## Section Seventeen: Policy Management Operations

### 17.1 Viewing the List of Policies

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy list minio-admin

---

### 17.2 Viewing Policy Details

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy info minio-admin policy-demo-readwrite

---

### 17.3 Detaching a User's Policy

Example: Detaching app01-user's policy.

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy detach minio-admin policy-demo-app01-readwrite --user app01-user

Verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user disable minio-admin app01-user

Note:

    Disabling a user is safer than removing them.
    It can quickly block access.
    Decide whether to delete or recreate the account after confirming the business switch.

---

### 19.3 Creating New Users or Keys

Recommended approach:

    Create a new user.
    Bind them with the same or more restrictive Policy.
    Adjust the business configuration.
    Verify that the service is functioning properly.
    Monitor the logs.
    Keep the disabled old user active for a while or delete it later.

Example of creating a new user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user add minio-admin app01-user-v2 'App01UserV2@123456'

Binding a Policy:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy attach minio-admin policy-demo-app01-readwrite --user app01-user-v2

---

### 19.4 Troubleshooting Abnormal Access

Check these areas:

    Nginx access.log
    MinIO logs
    Changes to Bucket objects
    Whether there are large amounts of downloads or deletions
    Any unusual uploads
    Access from unfamiliar IPs
    Unexplained increases in storage usage

Example commands:

    tail -f /var/log/nginx/access.log
    tail -f /var/log/nginx/error.log
    docker logs minio

Viewing objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-admin/policy-demo

---

### 19.5 Summary of Emergency Procedures

Key leakage handling steps:

    1. Identify the leaked AccessKey.
    2. Retrieve the corresponding user and Policy details.
    3. Immediately disable the user account.
    4. Review access logs and object changes.
    5. Create a new user or generate a new key.
    6. Bind them with minimal permission policies.
    7. Adjust the business configuration.
    8. Verify that services are functioning correctly.
    9. Monitor for any issues for a period of time.
    10. Eventually delete the old account or keep it disabled.
    11. Investigate the cause of the leakage.
    12. Strengthen Secret key management practices.

---

## Chapter Twenty: Methods for Setting Permissions in Production Environments

### 20.1 Separation by Business Areas

Recommendation:

| Business | Bucket | User |
|---|---|---|
| app01 | app01-uploads-prod | app01-prod |
| app02 | app02-uploads-prod | app02-prod |
| backup | backup-prod | backup-prod |
| logs | logs-archive-prod | logs-prod |
| devops | devops-artifacts-prod | devops-prod |

Benefits:

    Clearer permission boundaries.
    Easier tracking of resource usage and alerts.
    Reduced risk from key breaches affecting specific areas.
    Simplified cleanup processes after services are decommissioned.

---

### 20.2 Separation by Environments

Do not share keys between production and testing environments.

Recommendation:

    app01-dev
    app01-test
    app01-prod

Corresponding Buckets:

    app01-uploads-dev
    app01-uploads-test
    app01-uploads-prod

Reasons:

    Prevent test operations from affecting production.
    Ensure that developers do not have access to production keys.
    Facilitate independent rotation of keys.
    Improve auditability.

---

### 20.3 Separation by Permissions

Even within the same business area, permissions can be further refined:

| User | Permissions |
|---|---|
| app01-reader | Read-only access |
| app01-writer | Upload🔤 docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-admin/policy-demo

Delete other-demo:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-admin/other-demo

If other-demo is not empty, clear it first before deleting.

---

### 23.2 Remove Experimental Users

It is recommended to disable the users before removing them:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user disable minio-admin policy-readwrite-user

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user remove minio-admin policy-readwrite-user

Remove app01-user:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user remove minio-admin app01-user

Remove app01-user-v2:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user remove minio-admin app01-user-v2

If policy-readonly-user has already been deleted, ignore any error messages.

---

### 23.3 Remove Experimental Policies

Delete them:

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

## Chapter Twenty-Four: Production Permission Governance Checklist

### 24.1 User Checks

| Check Item | Requirement | Result |
|---|---|---|
| Root User | Should be used only for management | |
| Business Users | Each business should have independent users | |
| Environmental Isolation | Different environments (dev/test/prod) should use separate users | |
| Departed Staff | Accounts should be cleared promptly | |
| Abandoned Businesses | Users should be disabled or deleted | |
| User Naming | Names should reflect the business and environment | |

---

### 24.2 Policy Checks

| Check Item | Requirement | Result |
|---|---|---|
| Minimum Permissions | Only necessary actions should be granted | |
| Bucket Scope | Only necessary buckets should be authorized | |
> Prefix Scope > Prefixes should be limited when needed | |
> Delete Permission > The DeleteObject action should be granted with caution | |
> Management Permissions > Ordinary business users should not have them | |
> Wildcard Permissions > The Resource: * pattern should be used sparing16. In the production environment, it is necessary to establish audit lists for users, policies, keys, and operations.
17. Subsequently, further learning on MinIO data protection will be conducted: Erasure Coding, as well as recovery from node failures and disk failures.

---

## Chapter 27: Reference Documents

MinIO Identity and Access Management:

    https://min.io/docs/minio/linux/administration/identity-access-management.html

MinIO User Management:

    https://min.io/docs/minio/linux/administration/identity-access-management/minio-user-management.html

MinIO Policy Management:

    https://min.io/docs/minio/linux/administration/identity-access-management/policy-based-access-control.html

MinIO mc Admin User:

    https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-user.html

MinIO mc Admin Policy:

    https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-policy.html

MinIO mc Alias:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-alias.html

MinIO mc Cp:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-cp.html

AWS S3 Policy Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-policy-language-overview.html

AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html