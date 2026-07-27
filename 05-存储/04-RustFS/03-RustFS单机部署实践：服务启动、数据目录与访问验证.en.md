# Practical Single-Node Deployment of RustFS: Service Startup, Data Directory Setup, and Access Verification

Recommended Path: 05-Storage/04-RustFS/03-Practical Single-Node Deployment of RustFS: Service Startup, Data Directory Setup, and Access Verification.md

Tags: #RustFS #Object Storage #S3 #Docker #Single-Node Deployment #Bucket #Object #mc #AWSCLI #Data Persistence #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the third part of the RustFS series, focusing on the practical single-node Docker deployment of RustFS.

What has been covered previously includes:

    01-RustFS Basics: S3-Compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Understanding Single-Node and Cluster Modes

This article delves into the actual single-node implementation of RustFS, covering the following key points:

    Selection of a fixed RustFS image version
    Pulling and synchronizing the RustFS image to Alibaba Cloud's image repository
    Preparing the single-node data directory
    Setting data directory permissions
    Starting the RustFS Docker container
    Verifying API and Console ports
    Checking container logs
    Accessing health check endpoints
    Logging in to the Web Console
    Configuring the mc client
    Creating a Bucket
    Uploading and downloading Objects
    Verifying data persistence after container restarts
    Troubleshooting common issues
    Cleaning up the experiment environment

The deployment mode used in this article is:

    SNSD: Single Node Single Disk
    That is, a single-node, single-disk configuration.

The goal of this article is not to achieve high availability for production use but rather to ensure that RustFS can function at a minimum viable level:

    The service can be started.
    Ports can be accessed.
    Clients can connect successfully.
    Buckets can be created.
    Objects can be uploaded and downloaded.
    Data persistence is ensured.
    Issues can be identified and resolved.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Deploy a single-node RustFS service using Docker.
2. Understand how data directories are organized in the single-node mode of RustFS.
3. Comprehend why persistence of the host directory is necessary for the RustFS container.
4. Grasp the relationship between the running user and directory permissions within the RustFS container.
5. Learn how to fix a specific RustFS image version.
6. Synchronize the official RustFS image to your own Alibaba Cloud image repository.
7. Start the RustFS container and map API/Console ports correctly.
8. View RustFS container logs.
9. Verify the normal operation of the API through the health check endpoint.
10. Access the RustFS Console.
11. Configure RustFS aliases using the mc client.
12. Create a Bucket.
13. Upload and download Objects.
14. Verify the integrity of files before and after uploading.
15. Ensure that data is not lost after the container restarts.
16. Resolve issues such as port conflicts, permission errors, key issues, and client access failures.
17. Lay the foundation for future multi-node cluster deployments and HTTPS reverse proxy setups.

---

## III. Official Deployment Requirements

The official RustFS Docker installation documentation states:

    RustFS Docker single-node deployment is suitable for local testing and small-scale scenarios.
    It is recommended to use a Docker version >= 20.10.
    The default service port is usually 9000.
    The Console port is typically 9001.
    The data directory must be mounted on the host path.
    The running user within the RustFS container is rustfs, with a UID of 10001.
    If the host directory is mounted into the container using the -v option, the corresponding permissions must be granted to UID 10001.
    The API status can be checked through the /health endpoint.
    The Console can be accessed via http://<IP>:9001/rustfs/console.

This article will follow these requirements for implementation.

---

## IV. Experimental Environment Setup

### 4.1 Node Planning

A single-node setup is used in this experiment:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS single-node Docker node |

For testing purposes, a client node can also be used:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.55 | rustfs-client | For mc/AWS CLI tests |

If you don't have a separate client node yet, you can perform mc tests directly on rustfs-node01.

---

### 4.2 Operating System

The default operating system used is:

    Ubuntu Server If accessing the official Docker repository domestically is unstable, you can use a configured domestic apt repository or perform offline installation.

The focus of this document is on RustFS; therefore, detailed instructions for Docker installation will not be covered here.

---

### 5.4 Checking Port Occupancy

Execute:

    ss -lntp

Pay special attention to:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'

If there is any output, it indicates that the ports are already in use.

Action Steps:

    Identify the process occupying the ports.
    Stop any conflicting services.
    Alternatively, modify the port mapping for RustFS.
    Do not blindly terminate production processes.

To view processes:

    ps -ef | grep <PID>

---

### 5.5 Checking the Disk

Execute:

    df -hT
    lsblk -f

Check the /data directory:

    df -hT /data || true
    ls -ld /data || true

If /data does not exist:

    mkdir -p /data

Verify again:

    df -hT /data

If /data is still on the system disk:

    You can proceed with the experiment. However, it is not recommended to use the system disk for production purposes.

---

## VI. Image Pulling and Synchronization

### 6.1 Setting Version Variables

Execute:

    export RUSTFS_VERSION="1.0.0-alpha.99"
    export RUSTFS_SRC_IMAGE="rustfs/rustfs:${RUSTFS_VERSION}"
    export RUSTFS_DST IMAGE="registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:${RUSTFS_VERSION}"

Verify:

    echo ${RUSTFS_SRCIMAGE}
    echo ${RUSTFSDst IMAGE}

---

### 6.2 Pulling from the Official Image

If the network is functioning normally:

    docker pull ${RUSTFS_SOURCE_IMAGE}

If you are using an mdocker/Docker image acceleration address, you can pull the image using the available prefix, for example:

    docker pull docker.m.daocloud.io/${RUSTFS_SRC IMAGE}

Then re-tag it back to the official image name:

    docker tag docker.m.daocloud.io/${RUSTFS_SOURCE_IMAGE} ${RUSTFS_SRC_IMAGE}

Note:

    The acceleration address depends on the currently available services. This document records the method but does not bind to any specific third-party service. For production, it is recommended to synchronize images to your own Harbor or Alibaba Cloud image repository.

---

### 6.3 Viewing Images

Execute:

    docker images | grep rustfs

Expected output:

    rustfs/rustfs   1.0.0-alpha.99

---

### 6.4 Logging in to the Alibaba Cloud Image Repository

Execute:

    docker login registry.cn-hangzhou.aliyuncs.com

Enter:

    Username
    Password or Access Token

Security Reminder:

    Do not share the repository password in public notes.
    Do not save the docker login output in a public repository.
    For production, it is recommended to use a dedicated bot account or access token.

---

### 6.5 Tagging Images to the Alibaba Cloud Repository

Execute:

    docker tag ${RUSTFS_SRC_IMAGE} ${RUSTFS_DST IMAGE}

Verify:

    docker images | grep rustfs

Expected output:

    rustfs/rustfs:1.0.0-alpha.99
    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

---

### 6.6 Pushing Images to the Alibaba Cloud Repository

Execute:

    docker push ${RUSTFS_DST IMAGE}

For subsequent experiments, use by default:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

---

### 6.7 Recording Image Source Information

For this experiment, the image sources are:

    Source Image: rustfs/rustfs:1.0.0-alpha.99
    Target Image: registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99
    Synchronization Method: docker pull -> docker tag -> docker push
    Purpose: RustFS single-machine Docker experiment and subsequent multi-node experiments
    Reason for Version Selection: Using a fixed alpha version to avoid differences in commands, parameters, functions, or interfaces due to updates
    Production Note: This version is not recommended for production use. Production environments must re-evaluate the stability and security fixes of this version.

---

## VII. Preparing the Data Directory

### 7.1 Creating the Data Directory

Execute on rustfs-node01:

    mkdir -p /data/rustfs

Verify:

    ls -ld /data/rustfs
    df -h9000 is listening.
9001 is listening.

---

## IX. API Health Check

### 9.1 Local Check

Execute:

    curl -i http://127.0.0.1:9000/health

or:

    wget -Sq http://127.0.0.1:9000/health -O -

Expected response:

    HTTP/1.1 200 OK

---

### 9.2 Check via Node IP

Execute:

    curl -i http://10.0.0.51:9000/health

Expected response:

    HTTP/1.1 200 OK

---

### 9.3 Client Node Check

If there is a rustfs-client node, execute on 10.0.0.55:

    curl -i http://10.0.0.51:9000/health

If it fails, check:

    Whether the RustFS container is running.
    Whether port 9000 is being listened on.
    Whether the firewall permits access.
    Whether the two machines are networkedly connected.
    Whether the incorrect IP address was entered.

---

## X. Accessing the RustFS Console

### 10.1 Browser Access

Access:

    http://10.0.0.51:9001/rustfs/console

Login credentials:

    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456

Note:

    HTTP can be used in the experimental environment.
    HTTPS must be used for external production access.
    The management interface should not be exposed to the public network.
    Nginx will be configured later to provide a unified HTTPS entry point.

---

### 10.2 Troubleshooting If the Console Cannot Be Accessed

Check the container:

    docker ps | grep rustfs-single

Check the logs:

    docker logs rustfs-single --tail=200

Check the ports:

    ss -lntp | grep ':9001'

Check the configuration parameters:

    docker inspect rustfs-single | grep -i console -C 3

Common issues:

    The console is not enabled.
    Port 9001 is not mapped.
    Firewalls are blocking access.
    The access path is incorrect.
    The container failed to start.

---

## XI. Installing the mc Client

### 11.1 Using Docker Version of mc

If mc is not installed on your system, you can directly use the MinIO mc image.

Image available at:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Create a configuration directory for mc:

    mkdir -p /data/rustfs/mc-config

Test mc:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

---

### 11.2 Using Binary mc

If you have access to the external internet, you can install the binary version of mc.

Example for Linux amd64:

    curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
    chmod +x mc
    mv mc /usr/local/bin/mc

Verify installation:

    mc --version

If the internet connection is unstable, it is recommended to continue using the Docker version of mc.

---

## XII. Configuring mc to Access RustFS

### 12.1 Setting an Alias

Using Docker version of mc:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@123456'

Check the alias:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 12.2 If Using Local mc

Execute:

    mc alias set rustfs http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@      ls rustfs/app-uploads

---

### 14.3 Uploading Directories

Uploading logs directory:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp/rustfs-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/logs rustfs/app-uploads/

Checking:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs/app-uploads

---

### 14.4 Uploading Large Files

Uploading a 100Mi file:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp/rustfs-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /test/file-100m.bin rustfs/app-uploads/file-100m.bin

Checking the object:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      stat rustfs/app-uploads/file-100m.bin

---

### 14.5 Downloading Objects

Creating a download directory:

    mkdir -p /tmp/rustfs-download

Downloading hello.txt:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp/rustfs-download:/download \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp rustfs/app-uploads/hello.txt /download/hello.txt

Downloading a large file:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp/rustfs-download:/download \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp rustfs/app-uploads/file-100m.bin /download/file-100m.bin

Checking:

    ls -lh /tmp/rustfs-download

---

### 14.6 Verifying File Consistency

Performing:

    sha256sum /tmp/rustfs-test/hello.txt /tmp/rustfs-download/hello.txt
    sha256sum /tmp/rustfs-test/file-100m.bin /tmp/rustfs-download/file-100m.bin

If the hashes are identical, it indicates that the content of the uploaded and downloaded objects is consistent.

---

## Section Fifteen: Verifying Data Persistence After Container Restart

### 15.1 Restarting the Container

Performing:

    docker restart rustfs-single

Checking:

    docker ps | grep rustfs-single
    docker logs rustfs-single --tail=100

---

### 15.2 Checking the API

Performing:

    curl -i http://10.0.0.51:9000/health

Expected response:

    HTTP/1.1 200 OK

---

### 15.3 Rechecking the Bucket and Objects

Performing:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs

Checking objects:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs/app-uploads

Expected results:

    app-uploads should still exist.
    hello.txt should still exist.
    logs/app.log should still exist.
    fileIs the Endpoint URL correct?

If you encounter:

AccessDenied

Check:

- Whether the account and password are correct.
- Whether the Bucket exists.
- Whether the permissions are sufficient.

---

## Section Eighteen: Error Key Access Verification

### 18.1 Testing with an Incorrect SecretKey

Execute:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-wrong http://10.0.0.51:9000 rustfsadmin 'WrongPassword123'

Try to access:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-wrong

Expected outcome:

    Authentication failure.
    AccessDenied.
    SignatureDoesNotMatch.
    Or similar authentication errors.

Explanation:

    An incorrect key will not allow access to the object storage.
    This is a basic security check.

---

## Section Nineteen: Common Issues Troubleshooting

### 19.1 Container Failure to Start

Check:

    docker ps -a | grep rustfs-single
    docker logs rustfs-single --tail=200

Common causes:

- Port conflict.
- Insufficient permissions on the data directory.
- The image does not exist.
- Incorrect startup parameters.
- The SecretKey is too weak or does not meet format requirements.
- The data directory is not writable.

Actions to take:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'
    ls -ld /data/rustfs
    chown -R 10001:10001 /data/rustfs
    docker logs rustfs-single --tail=200

---

### 19.2 Permission Denied

Symptom:

    "Permission denied" appears in the docker logs.

Troubleshoot:

    ls -ld /data/rustfs
    id
    docker inspect rustfs-single | grep -i user -C 3

Actions to take:

    chown -R 10001:10001 /data/rustfs
    docker restart rustfs-single

Explanation:

    The RustFS container is not running under the root user of the host.
    The mount directory on the host must allow UID 10001 to write.

---

### 19.3 Inability to Access Port 9000

Troubleshoot:

    docker ps | grep rustfs-single
    ss -lntp | grep ':9000'
    curl -i http://127.0.0.1:9000/health
    curl -i http://10.0.0.51:9000/health
    docker logs rustfs-single --tail=100

Possible causes:

- The container is not running.
- The port is not mapped.
- Firewall restrictions.
- RustFS is not listening properly.
- Incorrect access IP address.

---

### 19.4 Unable to Log in to the Console

Troubleshoot:

    http://10.0.0.51:9001/rustfs/console
    docker logs rustfs-single --tail=100
    ss -lntp | grep ':9001'

Check:

- Whether the AccessKey is rustfsadmin.
- Whether the SecretKey is RustFSAdmin@123456.
- Whether old container parameters are being used.
- Whether environment variables have been changed without recreating the container.
- Whether the Console is enabled.

If you need to change the AccessKey/SecretKey, it is recommended to:

- Back up your data first.
- Stop the container.
- Confirm whether it will affect existing accounts.
- Re-create the container.
- Do not randomly change access keys in a production environment.

---

### 19.5 Failure to Set mc alias

Troubleshoot:

    curl -i http://10.0.0.51:9000/health
    docker run --rm -v /data/rustfs/mc-config:/root/.mc registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z alias list
    docker logs rustfs-single --tail=100

Common causes:

- Incorrect Endpoint.
If you only need to clean up the experimental alias:

    rm -rf /data/rustfs/mc-config

If you plan to continue with RustFS experiments in the future, it is recommended to keep it.

---

## 23. Standards for Completing This Practical Guide

After completing this guide, you should have achieved at least the following:

| Item | Standard |
|---|---|
| Image Version | Use the fixed version rustfs/rustfs:1.0.0-alpha.99 |
| Image Synchronization | Successfully synchronized to registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99 |
| Data Directory | The /data/rustfs directory has been created |
| Directory Permissions | UID 10001 should have write access |
| Container | rustfs-single is running |
| API Port | 10.0.0.51:9000 is accessible |
| Console Port | 10.0.0.51:9001 is accessible |
| Health Check | /health returns a status code of 200 |
| mc alias | The rustfs alias has been configured successfully |
| Bucket | app-uploads has been created successfully |
| Object Upload | Files hello.txt, logs, and file-100m.bin have been uploaded successfully |
| Object Download | Downloads were successful |
| Data Verification | sha256sums match |
| Restart Verification | After restarting docker, the Bucket and Objects still exist |
| Troubleshooting Ability | Able to handle issues with ports, permissions, authentication, and access failures |

---

## 24. Interview Answer Guidelines

If you are asked in an interview:

    How would you deploy RustFS on a single-node Docker machine? What checks would you perform?

You can answer as follows:

    Deploying RustFS on a single-node Docker machine falls under the SNSD model, which is a single-node, single-disk configuration. This setup is mainly suitable for learning purposes and basic feature verification and is not suitable for high-availability production environments.
    I would first fix the image version to rustfs/rustfs:1.0.0-alpha.99 instead of using the latest version. In a domestic network environment, I would pull the image from the official repository and then tag it to my own Alibaba Cloud mirror repository, for example, registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99. This ensures that future deployments use the same mirror, making the experiments reproducible.
    Before deployment, I would check Docker, ports, disk, and time synchronization. The default API port for RustFS containers is 9000, and the commonly used Console port is 9001. I would place the data directory on the host machine at /data/rustfs and mount it into the container using -v /data/rustfs:/data to prevent data from being stored only in the temporary layer of the container. Since the default user UID running in the RustFS container is 10001, I would need to use chown -R 10001:10001 /data/rustfs on the host machine directory to avoid permission issues.
    After starting the container, I would check docker ps, docker logs, and ss -lntp. I would also access http://10.0.0.51:9000/health to verify the API health status and http://10.0.0.51:9001/rustfs/console to test the Console functionality.
    For client verification, I would use mc to configure aliases and create a Bucket, such as app-uploads. Then, I would upload files like hello.txt, small directories, and a 100Mi test file, and download them again to perform sha256sum verifications. Finally, I would restart the container and check again using mc ls to ensure that the Bucket and Objects still exist, confirming that the data has been successfully persisted on the host machine directory.
    In case of troubleshooting, if the container fails to start, I would first check docker logs; if port access fails, I would check ss -lntp, firewalls, and curl /health; if uploads fail, I would check the Bucket, permissions, data directory, and disk space; if Console login fails, I would verify the AccessKey, SecretKey, and whether the Console is enabled.
    In production, I would not use a single-node mode for critical services. The single-node mode can only demonstrate that RustFS's basic functions are usable. Further steps such as setting up multi-node clusters, implementing unified HTTPS access, managing permissions, monitoring and alerting, conducting fault drills, and performing performance testing are necessary.

---

## 25. Summary of This Article