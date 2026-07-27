# MinIO Client Tool: mc Configuration, Bucket Management, and Object Operations

Recommended Path: 05-Storage/02-MinIO/06-MinIO Client Tool: mc Configuration, Bucket Management, and Object Operations.md

Tags: #MinIO #mc #Object Storage #S3 #Bucket #Object #Client Tool #Upload and Download #Advanced SRE #Production Ops

---

## I. Document Overview

This article is the sixth in the MinIO series, focusing on learning how to configure the official MinIO client tool, mc, as well as manage Buckets and perform object operations.

Previously covered topics include:

- Basics of MinIO Object Storage
- Single-machine single-disk deployment
- Single-node multi-disk deployment
- 4-node multi-disk distributed cluster deployment
- Design of internal HTTP and external HTTPS access points
- Nginx HTTPS unified entry configuration

This article now moves on to the practical operations aspect of MinIO.

mc is one of the most commonly used command-line tools in MinIO operations, similar to kubectl in the object storage field. It can be used for:

    Configuring MinIO connection aliases
    Viewing cluster information
    Creating Buckets
    Deleting Buckets
    Uploading objects
    Downloading objects
    Checking object metadata
    Viewing Bucket capacity
    Recursively listing objects
    Deleting objects
    Synchronizing directories
    Performing mirror backup and migration
    Managing users, Policies, lifecycle rules, versions, and other advanced features

This article will first cover the basic and frequently used functions:

    mc alias
    mc admin info
    mc mb
    mc ls
    mc cp
    mc stat
    mc du
    mc find
    mc rm
    mc rb
    Basic usage of mc mirror

Permissions, users, and Policies will be discussed in detail in the next article:

    07-MinIO Permission Management: Users, Policies, AccessKey, and SecretKey.md

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the role of mc in MinIO operations.
2. Run mc using Docker without installing it locally.
3. Configure a MinIO backend HTTP alias.
4. Set up an HTTPS unified entry alias for MinIO.
5. Use mc to view MinIO cluster information.
6. Create Buckets using mc.
7. Upload files to Buckets with mc.
8. Download files from Buckets using mc.
9. Check object metadata using mc.
10. View Bucket capacity and the number of objects.
11. Recursively list objects using mc.
12. Delete individual objects, Prefix objects, and Buckets.
13. Perform basic directory synchronization using mc mirror.
14. Master common troubleshooting methods for mc.
15. Understand the potential risks associated with using mc in production environments.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This article continues from the previous MinIO distributed cluster setup:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc Client |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entry |

---

### 3.2 Access Points

Direct backend access point:

    http://10.0.0.41:9000

HTTPS unified access point:

    https://s3.minio.local

Console access point:

    https://console.minio.local

Note:

    mc connects to the S3 API entry point.
    It should connect to port 9000 or the HTTPS API domain name.
    Do not configure the 9001 Console address for mc.

---

### 3.3 Image Versions

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Source images:

    minio/mc:RELEASE.2025-04-16T18-13-26Z
    minio/minio:RELEASE.2025registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

If a self-signed certificate is used for the experiment:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

Note:

    The --insecure option is only suitable for experiments using self-signed certificates. In production, trusted certificates should be used, and certificate verification should not be bypassed for extended periods.

---

### 6.3 Viewing the Alias List

Run the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

If you see the following aliases:

    minio-backend
    minio-https

It indicates that the configuration is successful.

---

### 6.4 Deleting Aliases

If the configuration is incorrect, you can delete the aliases using the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias remove minio-backend

After deletion, you can reconfigure the aliases as needed.

---

## VII. Viewing MinIO Cluster Information

### 7.1 Viewing Backend Cluster Information

Run the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-backend

Pay attention to the following fields:

    MinIO version
    Network
    Drives
    Pool
    Used
    Online
    Offline
    Healing
    Errors

---

### 7.2 Viewing HTTPS Entry Cluster Information

If a self-signed certificate is used:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-https

If an official certificate is used:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-https

---

### 7.3 The Value of Using admin info for Troubleshooting

admin info can help you determine the following:

    Whether the cluster is accessible.
    Whether the nodes are online.
    Whether the disks are online.
    Whether the cluster capacity is normal.
    Whether there are any offline drives.
    Whether there is any healing process in progress.
    Whether the reverse proxy entrance is functioning correctly.
    Whether the AccessKey and SecretKey are correct.

If using admin info yields no results, first check the following:

    Whether the alias endpoint is correct.
    Whether the 9001 Console is being used incorrectly.
    Whether the MinIO backend is healthy.
    Whether Nginx is running properly.
    Whether the certificate is trustworthy.
    Whether the AccessKey and SecretKey are correct.

---

## VIII. Bucket Management

### 8.1 Viewing the Bucket List

Run the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https

If you are connecting to the backend HTTP:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.ali```markdown
--insecure cp /demo/upload/hello.txt minio-https/mc-demo/hello.txt

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https/mc-demo

---

### 9.3 Upload to a Specified Prefix

Upload the log file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/upload/logs/nginx/2026/04/28/access.log minio-https/mc-demo/logs/nginx/2026/04/28/access.log

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https/mc-demo/logs/nginx/2026/04/28/

Explanation:

    logs/nginx/2026/04/28/access.log is the Object Key.
    The prefix logs/nginx/2026/04/28/ indicates the directory structure.
    In object storage, this essentially serves as a key prefix.

---

### 9.4 Upload Large Files

Upload a 100M file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/upload/file-100m.bin minio-https/mc-demo/file-100m.bin

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure stat minio-https/mc-demo/file-100m.bin

If the large file upload fails, check the following key factors:

    Nginx's client_max_body_size setting
    Nginx's proxy_request_buffering and proxy_read_timeout settings
    The readiness status of the MinIO backend
    Disk capacity
    Network stability

---

### 9.5 Recursively Upload Directories

Upload the entire upload directory:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp --recursive /demo/upload/ minio-https/mc-demo/full-upload/

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-https/mc-demo/full-upload/

---

## Section X: Object Download Operations

### 10.1 Download a Single Object

Download hello.txt:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-https/mc-demo/hello.txt /demo/download/hello-download.txt

View:

    cat /tmp/minio-mc-demo/download/hello-download.txt

---

### 10.2 Download Objects with a Prefix

Download the log object:

    mkdir -p /tmp/minio-mc-demo/download/logs

    docker run --rm \
     ```bash
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure stat minio-https/mc-demo/hello.txt
```

**Key Points:**  
- The `stat` command can be used to check if an object exists, its size, and whether the upload was complete.

---

### 11.4 Viewing Object Contents

For text objects, you can directly view them using:

```bash
docker run --rm \
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure cat minio-https/mc-demo/hello.txt
```

However, using `cat` on large objects is not recommended.

---

## Section 12: Checking Capacity and Object Sizes

### 12.1 Checking Bucket Capacity

Run the following command:

```bash
docker run --rm \
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure du minio-https/mc-demo
```

---

### 12.2 Checking Prefix Capacity

To check the capacity of logs prefixes, run:

```bash
docker run --rm \
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure du minio-https/mc-demo/logs/
```

To check the capacity of full-upload prefixes, run:

```bash
docker run --rm \
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure du minio-https/mc-demo/full-upload/
```

---

### 12.3 Operational Value of Capacity Checking

Checking capacity is useful for:

- Monitoring the growth trend of Buckets.
- Identifying any abnormalities in data uploads.
- Detecting unusual increases in specific Prefixes.
- Troubleshooting object storage capacity alerts.
- Conducting capacity assessments before migrations or backups.

**Production Recommendations:**
- Ensure that each business Bucket has a designated owner.
- Maintain capacity statistics for all Buckets.
- Implement lifecycle policies for large Buckets.
- Establish backup strategies for critical Buckets.
- Set up alerts for any unexpected capacity increases.

---

## Section 13: Object Deletion

### 13.1 Deleting a Single Object

To delete the `hello.txt` file, run:

```bash
docker run --rm \
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure rm minio-https/mc-demo/hello.txt
```

To confirm the deletion, run:

```bash
docker run --rm \
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure ls minio-https/mc-demo
```

---

### 13.2 Deleting Objects under a Prefix

**High-Risk Warning:**  
The `--recursive` and `--force` options will delete objects recursively and without confirmation, respectively. In a production environment, always confirm the contents of Buckets and Prefixes before deleting. Also, make sure to have backups in place.

To delete the logs prefix, run:

```bash
docker run --rm \
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure rm --recursive --force minio-https/mc-demo/logs/
```

To confirm the deletion, run:

```bash
docker run --rm \
-v /data/minio/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.202```markdown
--insecure find minio-https/mirror-demo

---

### 14.5 Synchronizing Buckets to Local Directories

Create a local recovery directory:

    mkdir -p /tmp/minio-mc-demo/mirror-restore

Execute the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror minio-https/mirror-demo/ /demo/mirror-restore/

View the contents:

    tree /tmp/minio-mc-demo/mirror-restore

Verify the files:

    cat /tmp/minio-mc-demo/mirror-restore/app/config.txt
    cat /tmp/minio-mc-demo/mirror-restore/logs/app.log

---

### 14.6 Differences Between `mirror` and `cp`

| Command          | Suitable Scenarios                |
|-----------------|----------------------------------------|
| mc cp            | Uploading or downloading individual files   |
| mc cp --recursive    | Recursively copying directories         |
| mc mirror        | Synchronizing directories or Buckets      |
| mc mirror --watch     | Continuous synchronization (use with caution) |
| mc mirror --remove    | Deleting objects on the target if they don't exist | 

Production Reminder:

    The `mc mirror --remove` command is highly risky. It may delete data on the target side. Always perform a dry-run first or verify it in a test environment before using it in production. Ensure that logging and verification are implemented for migration and synchronization tasks.

---

## Section 15: Simplifying Command Usage

### 15.1 Defining Temporary Alias Functions

If running `docker run` frequently results in long commands, you can define temporary functions within the current shell session:

    export MC_IMAGE=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
    export MC_CONFIG=/data/minio/mc-config
    export MC_WORKDIR=/tmp/minio-mc-demo

    Create a function `mcx()`:

    mcx() {
      docker run --rm \
        -v ${MC_CONFIG}:/root/.mc \
        -v ${MC_WorkDIR}:/demo \
        ${MC_IMAGE} "$@"
    }

If using self-signed certificates, you can call it like this:

    mcx --insecure ls minio-https

To create a new Bucket:

    mcx --insecure mb minio-https/simple-demo

To upload a file:

    mcx --insecure cp /demo/upload/hello.txt minio-https/simple-demo/hello.txt

To view the contents:

    mcx --insecure ls minio-https/simple-demo

Note:

    These functions are only valid within the current shell session. Redefine them if you close the terminal. They are useful for reducing repetitive input during experimentation.

---

### 15.2 Production Recommendations

In production environments, it is recommended to:

    Install a fixed version of `mc` on the management server.
    Encapsulate related operations into internal maintenance scripts.
    Avoid explicitly specifying business-related SecretKeys in commands.
    Use dedicated maintenance accounts for these tasks.
    Always require double confirmation before executing high-risk deletion commands.
    Keep detailed operation logs.

---

## Section 16: Troubleshooting Common Issues

### 16.1 Failure to Set `mc alias`

Possible causes:

    The endpoint was entered incorrectly.
    Confusing the 9001 Console with the API.
    Incorrect AccessKey or SecretKey.
    The MinIO backend is not accessible.
    Nginx's 443 port is blocked.
    The self-signed certificate is not trusted.

Troubleshooting steps:

    Check using `curl -I http://10.0.0.41:9000/minio/health/live`.
    Verify with `curl -k -I https://s3.minio.local/minio/health/live`.
    Check `docker logs minio` for any error messages.
    Monitor `/var/log/nginx/error.log` for relevant errors.

---

### 16.2 `mc` Indicates That the Certificate Is Signed by an Unknown Authority

Reason:

    A self-signed certificate is being used, and the client does not trust it.

Solution for experimentation:

    Use the `--insecure` option.

For production:

    Replace it with a legitimate certificate or import it into your enterprise's CA system. Avoid using `--insecure` in production environments on a long-term basis.

---

### 16.3 Failure to Upload Using `mc`

Order      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-https/mc-demo

---

### 19.2 Deleting the mirror-demo Object

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-https/mirror-demo

Delete the Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-https/mirror-demo

---

### 19.3 Clearing the Local Test Directory

    rm -rf /tmp/minio-mc-demo

---

## Twenty: Interview Answer Strategies

If you are asked in an interview:

    What is the mc tool of MinIO commonly used for?

You can answer like this:

    The mc tool is the official command-line client for MinIO, which can be considered a widely used management tool in object storage operations. It allows you to configure alias connections to MinIO or other S3-compatible object storages, and then perform tasks such as Bucket management, object upload/download, capacity checking, metadata viewing, object deletion, directory synchronization, and mirror migration.
    In daily operations, I use `mc alias set` to configure connections for different environments, such as `minio-prod` and `minio-test`. I also use `mc admin info` to check the status of cluster nodes, disks, and capacity. To create a Bucket, I use `mc mb`. For uploading or downloading objects, I use `mc cp`. To view object metadata, I use `mc stat`. To check the capacity of a Bucket or Prefix, I use `mc du`. To search for objects, I use `mc find`. And for directory synchronization or cross-cluster migration, I use `mc mirror`.
    It is important to note that `mc` connects to MinIO's S3 API address, which is port 9000 or the HTTPS API entry point, not port 9001 for the Console. In a production environment, the root user should not be used for business operations; instead, independent users with minimal permission policies should be created.
    Additionally, commands like `mc rm --recursive --force`, `mc rb`, and `mc mirror --remove` are considered high-risk operations. Before executing them in production, it is essential to verify alias settings, Buckets, Prefixes, backup plans, approval processes, and rollback strategies to avoid accidentally deleting object data.

---

## Twenty-One: Summary of This Article

This article has provided practical guidance on using the MinIO mc client tool:

1. `mc` is the official command-line client for MinIO.
2. `mc` connects to the S3 API address, not the Console address.
3. Port 9000 is the API port, while port 9001 is for the Console.
4. `mc alias` is used to configure connections to MinIO.
5. `mc admin info` helps check cluster nodes, disks, and capacity status.
6. `mc mb` is used to create Buckets.
7. `mc rb` is used to delete Buckets.
8. `mc ls` lists Buckets or objects.
9. `mc cp` allows object upload and download.
10. `mc cp --recursive` copies directories recursively.
11. `mc stat` provides object metadata.
12. `mc cat` displays the content of text objects.
13. `mc du` shows the capacity of Buckets or Prefixes.
14. `mc find` searches for objects recursively.
15. `mc rm` deletes objects.
16. `mc mirror` performs basic directory synchronization and migration tasks.
17. The use of self-signed certificates in experiments is temporary and not recommended for production.
18. The root user should only be used for management purposes; independent AccessKeys and policies should be assigned for business operations.
19. Deleting Buckets, recursively deleting objects, and using `mc mirror --remove` are high-risk operations.
20. In the next article, we will explore MinIO's permission management: users, policies, AccessKeys, and SecretKeys.

---

## Twenty-Two: References

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO mc Alias Documentation