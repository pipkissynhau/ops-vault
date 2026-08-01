# RustFS Single-Node Deployment Practice: Service Startup, Data Directory, and Access Verification

Recommended path: 05-Storage/04-RustFS/03-RustFS Single-Node Deployment Practice: Service Startup, Data Directory, and Access Verification.md

Tags: #RustFS #ObjectStorage #S3 #Docker #UnitDeployment #Bucket #Object #mc #AWSCLI #DataSustainability #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the third article of the RustFS module, focusing on completing the RustFS single-node Docker deployment practice.

Previously completed:

    01-RustFS Basics: S3-Compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Single-Node and Cluster Mode Understanding

This article officially enters the RustFS single-node hands-on practice, focusing on:

    RustFS Image Fixed Version Selection
    RustFS Image Pulling and Synchronization to Alibaba Cloud Mirror Repository
    Single-Node Data Directory Preparation
    Data Directory Permission Settings
    Docker Startup of RustFS
    API Port and Console Port Verification
    Container Log Viewing
    Health Check Access
    Web Console Access
    mc Client Configuration
    Bucket Creation
    Object Upload and Download
    Data Persistence Verification After Container Restart
    Common Issue Troubleshooting
    Experiment Cleanup

The deployment mode of this article is:

    SNSD: Single Node Single Disk
    Single-Node Single Disk Mode

The goal of this article is not to achieve production high availability, but to complete the minimum viable RustFS loop:

    Service can start
    Ports can be accessed
    Clients can connect
    Bucket can be created
    Object can be uploaded and downloaded
    Data can be persisted
    Issues can be troubleshooted

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Deploy RustFS single-node service using Docker.
2. Understand the data directory planning for RustFS single-node mode.
3. Understand why RustFS containers need host directory persistence.
4. Understand the relationship between RustFS container runtime user and directory permissions.
5. Be able to fix the RustFS image version.
6. Be able to synchronize the RustFS official image to your Alibaba Cloud mirror repository.
7. Be able to start the RustFS container and map API/Console ports.
8. Be able to view RustFS container logs.
9. Be able to verify API health via the health interface.
10. Be able to access RustFS Console.
11. Be able to configure RustFS alias using mc.
12. Be able to create a Bucket.
13. Be able to upload an Object.
14. Be able to download an Object.
15. Be able to verify file consistency before and after upload.
16. Be able to verify data persistence after container restart.
17. Be able to troubleshoot port conflicts, permission issues, key errors, and client access failures.
18. Lay the foundation for subsequent multi-node cluster deployment and HTTPS reverse proxy.

---

## III. Official Deployment Highlights

The RustFS official Docker installation document states:

    RustFS Docker single-node deployment is suitable for local testing and small-scale scenarios.
    Docker version is recommended to be >= 20.10.
    Default service ports typically use 9000.
    Console port typically uses 9001.
    Data directory must be mounted to a host path.
    RustFS container runs as user rustfs with UID 10001.
    If a host directory is mounted into the container via -v, the directory permissions must be granted to UID 10001.
    API can be checked via /health interface.
    Console can be accessed via http://<IP>:9001/rustfs/console.

This article will implement these requirements.

---

## IV. Experiment Environment Planning

### 4.1 Node Planning

This article uses a single-node setup:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Single-Node Docker Node |

Clients can use:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.55 | rustfs-client | mc / AWS CLI Test Node |

If you don't have an independent client, you can also execute mc tests directly on rustfs-node01.

---

### 4.2 Operating System

Default system:

    Ubuntu Server 22.04.5 LTS

Optional system:

    Rocky Linux 9

This article focuses on Ubuntu 22.04.5.

---

### 4.3 RustFS Version Planning

This article uses a fixed version:

    rustfs/rustfs:1.0.0-alpha.99

Notes:

    This version belongs to the current RustFS alpha stage.
    Suitable for learning and experiment replication.
    Not recommended for direct production use.
    Production version selection must be combined with official Release Notes, known issues, S3 compatibility, stress test results, license, upgrade strategy, and recovery drills.

If you encounter runtime compatibility issues, you can evaluate the glibc version:

    rustfs/rustfs:1.0.0-alpha.99-glibc

This article defaults to:

    rustfs/rustfs:1.0.0-alpha.99

---

### 4.4 Image Repository Planning

Official source image:

    rustfs/rustfs:1.0.0-alpha.99

User's Alibaba Cloud image repository target image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

Image source recording method:

    First pull the official rustfs/rustfs:1.0.0-alpha.99 fixed version.
    Then re-tag it as registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99.
    Then push it to the Alibaba Cloud image repository.
    Subsequent experiments will default to using the user's own Alibaba Cloud image address.
    This reduces the probability of image pull failure in domestic network environments.

---

### 4.5 Port Planning

| Port | Purpose | Exposure Range |
|---|---|---|
| 9000 | RustFS S3 API | Internal Network Access |
| 9001 | RustFS Console | Internal Network Access |
| 443 | Future Nginx HTTPS Entry | To be used in 07 |
| 22 | SSH Maintenance | Only from maintenance sources |

This article directly exposes:

    10.0.0.51:9000
    10.0.0.51:9001

In subsequent 07, Nginx will be used to:

    https://s3.rustfs.local

---

### 4.6 Data Directory Planning

Host data directory:

    /data/rustfs

Container data directory:

    /data

Mounting Relationships:

    /data/rustfs:/data

Production Warnings:

    Experimental environments can use the current disk.
    Production environments recommend mounting /data to an independent data disk.
    Do not place RustFS data directory directly on the system disk for production data.
    Do not mix /data/rustfs with other business logs, caches, or temporary files.

---

## Five. Pre-deployment Checks

### 5.1 Check Host Information

Execute on rustfs-node01:

    hostname
    hostname -I
    cat /etc/os-release
    uname -a
    timedatectl

Key confirmations:

    Is the hostname rustfs-node01?
    Is the IP 10.0.0.51?
    Is the time synchronized?
    Does the system version match expectations?

---

### 5.2 Check Network

Execute:

    ip addr
    ip route
    ping -c 5 10.0.0.51

If there are client nodes:

    ping -c 5 10.0.0.55

Check hosts:

    getent hosts rustfs-node01

If no resolution, add:

    cat >> /etc/hosts <<'EOF'
    10.0.0.51 rustfs-node01
    10.0.0.55 rustfs-client
    EOF

---

### 5.3 Check Docker

Execute:

    docker version
    docker info

If Docker is not installed, Ubuntu can refer to:

    apt update
    apt install -y ca-certificates curl gnupg lsb-release

    install -m 0755 -d /etc/apt/keyrings

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
      -o /etc/apt/keyrings/docker.asc

    chmod a+r /etc/apt/keyrings/docker.asc

    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo ${UBUNTU_CODENAME:-$VERSION_CODENAME}) stable" \
      > /etc/apt/sources.list.d/docker.list

    apt update
    apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    systemctl enable --now docker

Check:

    systemctl status docker --no-pager
    docker version

Notes:

    If domestic access to Docker official source is unstable, use a pre-configured domestic apt source or offline installation.
    This document focuses on RustFS, not expanding Docker installation details.

---

### 5.4 Check Port Usage

Execute:

    ss -lntp

Key checks:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'

If output exists, the port is occupied.

Handling options:

    Confirm the occupying process.
    Stop conflicting services.
    Or modify RustFS mapped ports.
    Do not blindly kill production processes.

Check processes:

    ps -ef | grep <PID>

---

### 5.5 Check Disks

Execute:

    df -hT
    lsblk -f

Check /data:

    df -hT /data || true
    ls -ld /data || true

If /data does not exist:

    mkdir -p /data

Check:

    df -hT /data

If /data is still on the system disk:

    Experimentation can continue.
    Production is not recommended to directly use the system disk.

---

## Six. Image Pull and Synchronization

### 6.1 Set Version Variables

Execute:

    export RUSTFS_VERSION="1.0.0-alpha.99"
    export RUSTFS_SRC_IMAGE="rustfs/rustfs:${RUSTFS_VERSION}"
    export RUSTFS_DST_IMAGE="registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:${RUSTFS_VERSION}"

Check:

    echo ${RUSTFS_SRC_IMAGE}
    echo ${RUSTFS_DST_IMAGE}

---

### 6.2 Pull from Official Image

If network is normal:

    docker pull ${RUSTFS_SRC_IMAGE}

If using mdocker / Docker image acceleration address, pull with the current available acceleration prefix, for example:

    docker pull docker.m.daocloud.io/${RUSTFS_SRC_IMAGE}

Then re-tag back to the official image name:

    docker tag docker.m.daocloud.io/${RUSTFS_SRC_IMAGE} ${RUSTFS_SRC_IMAGE}

Notes:

    Acceleration address should be based on current available service.
    Notes record the method, not strongly bind to a third-party acceleration service.
    Production is recommended to synchronize to your own Harbor or Aliyun image repository.

---

### 6.3 Check Images

Execute:

    docker images | grep rustfs

Expected to see:

    rustfs/rustfs   1.0.0-alpha.99

---

### 6.4 Login to Aliyun Image Repository

Execute:

    docker login registry.cn-hangzhou.aliyuncs.com

Input:

    Username
    Password or access token

Security Reminder:

    Do not write repository passwords into public notes.
    Do not save docker login output to public repositories.
    Production recommends using dedicated robot accounts or access tokens.

---

### 6.5 Tag to Aliyun Repository

Execute:

    docker tag ${RUSTFS_SRC_IMAGE} ${RUSTFS_DST_IMAGE}

Check: /think

docker images | grep rustfs

Expected output:

    rustfs/rustfs:1.0.0-alpha.99
    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

---

### 6.6 Push to Alibaba Cloud Repository

Execute:

    docker push ${RUSTFS_DST_IMAGE}

Subsequent experiments will default use:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

---

### 6.7 Image Source Record

This experiment's image source:

    Source image: rustfs/rustfs:1.0.0-alpha.99
    Target image: registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99
    Synchronization method: docker pull -> docker tag -> docker push
    Purpose: RustFS single-node Docker experiment and subsequent multi-node experiment
    Version selection reason: Fixed alpha version to avoid command, parameter, feature, or interface differences caused by latest changes
    Production warning: This version is not recommended for direct production use. Production must re-evaluate version stability and security patch status

---

## VII. Prepare Data Directory

### 7.1 Create Data Directory

On rustfs-node01 execute:

    mkdir -p /data/rustfs

Check:

    ls -ld /data/rustfs
    df -hT /data/rustfs

---

### 7.2 Set Directory Permissions

RustFS container runtime user UID is typically 10001.

Therefore need to set host directory permissions:

    chown -R 10001:10001 /data/rustfs

Check:

    ls -ld /data/rustfs

Expected similar:

    drwxr-xr-x 2 10001 10001 ... /data/rustfs

Note:

    The host may not have a username corresponding to UID 10001.
    This does not affect container usage.
    If not set, Permission denied may occur.

---

### 7.3 Not Recommended to Use 777

Not recommended:

    chmod -R 777 /data/rustfs

Reason:

    Too broad permissions.
    Does not meet production security baseline.
    May mask actual permission issues.

Prefer using:

    chown -R 10001:10001 /data/rustfs

---

## VIII. Start RustFS Single-node Container

### 8.1 Define Account Password Variables

Execute:

    export RUSTFS_ACCESS_KEY="rustfsadmin"
    export RUSTFS_SECRET_KEY="RustFSAdmin@123456"

Security reminder:

    The account password in this document is only for experimentation.
    Production must use strong passwords.
    Do not commit plaintext keys to Git in production.
    Recommend using independent business AccessKey and avoid long-term use of management accounts.

---

### 8.2 Start Container

Use your own Alibaba Cloud image to start:

    docker run -d \
      --name rustfs-single \
      --restart=always \
      -p 9000:9000 \
      -p 9001:9001 \
      -v /data/rustfs:/data \
      -e RUSTFS_ACCESS_KEY="${RUSTFS_ACCESS_KEY}" \
      -e RUSTFS_SECRET_KEY="${RUSTFS_SECRET_KEY}" \
      -e RUSTFS_CONSOLE_ENABLE=true \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99 \
      --address :9000 \
      --console-enable \
      --access-key "${RUSTFS_ACCESS_KEY}" \
      --secret-key "${RUSTFS_SECRET_KEY}" \
      /data

Note:

    --name rustfs-single: Container name.
    --restart=always: Automatically restart after host or Docker reboot.
    -p 9000:9000: Expose S3 API port.
    -p 9001:9001: Expose Console port.
    -v /data/rustfs:/data: Mount host directory to container /data.
    RUSTFS_ACCESS_KEY: Access account.
    RUSTFS_SECRET_KEY: Access key.
    RUSTFS_CONSOLE_ENABLE=true: Enable Console.
    /data: RustFS data volume path.

Warning:

    Official documentation states environment variables and command-line parameters can be mixed, but command-line parameters have higher priority.
    For experimental clarity, this document retains both environment variables and command parameters.
    Production should choose one standardized method and avoid messy maintenance.

---

### 8.3 Check Container Status

Execute:

    docker ps | grep rustfs-single

Check full information:

    docker ps -a | grep rustfs-single

Expected:

    STATUS is Up

---

### 8.4 Check Logs

Execute:

    docker logs rustfs-single --tail=100

Continuous check:

    docker logs -f rustfs-single

Focus on:

    Whether listening on 9000.
    Whether Console is enabled.
    Whether permission denied occurs.
    Whether address already in use occurs.
    Whether access key/secret key related errors occur.
    Whether data directory anomalies occur.

---

### 8.5 Check Port Listening

Execute:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'

Expected:

    9000 is listening.
    9001 is listening.

---

## IX. API Health Check

### 9.1 Local Check

Execute:

    curl -i http://127.0.0.1:9000/health

Or:

    wget -Sq http://127.0.0.1:9000/health -O -

Expected: /think

HTTP/1.1 200 OK

---

### 9.2 Check via Node IP

Execute:

    curl -i http://10.0.0.51:9000/health

Expected:

    HTTP/1.1 200 OK

---

### 9.3 Client Node Check

If there is a rustfs-client node, execute on 10.0.0.55:

    curl -i http://10.0.0.51:9000/health

If failed, check:

    Whether the RustFS container is running.
    Whether port 9000 is listening.
    Whether the firewall allows traffic.
    Whether the two machines are network reachable.
    Whether the wrong IP was accessed.

---

## Ten. Access RustFS Console

### 10.1 Browser Access

Access:

    http://10.0.0.51:9001/rustfs/console

Login information:

    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456

Notes:

    HTTP can be used in experimental environments.
    HTTPS must be used for external access in production.
    The management entry should not be exposed to the public internet.
    Nginx HTTPS unified entry will be configured in section 07 later.

---

### 10.2 Troubleshooting Console Access Issues

Check container:

    docker ps | grep rustfs-single

Check logs:

    docker logs rustfs-single --tail=200

Check ports:

    ss -lntp | grep ':9001'

Check parameters:

    docker inspect rustfs-single | grep -i console -C 3

Common causes:

    Console is not enabled.
    Port 9001 is not mapped.
    Firewall interception.
    Incorrect access path.
    Container startup failure.

---

## Eleven. Install mc Client

### 11.1 Use Docker Version of mc

If mc is not available on this machine, you can directly use the MinIO mc image.

Images already used in the MinIO module:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Create mc configuration directory:

    mkdir -p /data/rustfs/mc-config

Test mc:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

---

### 11.2 Use Binary mc

If you can access the internet, you can install the binary mc.

Linux amd64 example:

    curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
    chmod +x mc
    mv mc /usr/local/bin/mc

Check:

    mc --version

If internet is unstable, it is recommended to continue using the Docker version of mc.

---

## Twelve. Configure mc to Access RustFS

### 12.1 Configure alias

Use Docker version of mc:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@123456'

Check alias:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 12.2 If Using Local mc

Execute:

    mc alias set rustfs http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@123456'
    mc alias list

---

### 12.3 Troubleshoot alias Failure

If failed, check:

    curl -i http://10.0.0.51:9000/health
    docker logs rustfs-single --tail=100
    ss -lntp | grep 9000

Common causes:

    RustFS API is not running.
    Endpoint is written incorrectly.
    AccessKey is written incorrectly.
    SecretKey is written incorrectly.
    Time is out of sync.
    Network is unreachable.
    Firewall is not allowing traffic.
    RustFS has compatibility issues with some client behaviors.

---

## Thirteen. Bucket Basic Operations

### 13.1 Create Bucket

Create test Bucket:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs/app-uploads

Check:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs

Expected:

    app-uploads

---

### 13.2 Re-create Bucket

Execute again:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs/app-uploads

May prompt:

    Bucket already exists

Notes:

    The bucket already exists.
    This is a normal prompt.

---

### 13.3 Bucket Naming Recommendations

Recommendation: /think

Use lowercase letters.  
Use numbers and hyphens.  
Do not use underscores.  
Do not use Chinese.  
Do not use spaces.  
Do not use long names.  
Distinguish Buckets by business.  

Example:  

    app-uploads  
    backups  
    logs-archive  
    ai-datasets  
    devops-artifacts  

---

## 14. Object Upload and Download Verification  

### 14.1 Prepare Test Files  

Create local test directory:  

    mkdir -p /tmp/rustfs-test  
    cd /tmp/rustfs-test  

Create small file:  

    echo "hello rustfs single node" > hello.txt  

Create configuration file:  

    cat > app.conf <<'EOF'  
    app_name=rustfs-demo  
    env=lab  
    endpoint=http://10.0.0.51:9000  
    EOF  

Create simulated log:  

    mkdir -p logs  
    echo "2026-04-28 INFO rustfs object upload test" > logs/app.log  

Create large file:  

    dd if=/dev/zero of=file-100m.bin bs=1M count=100  

Check:  

    ls -lh  

Calculate checksum:  

    sha256sum hello.txt app.conf logs/app.log file-100m.bin > sha256-before.txt  
    cat sha256-before.txt  

---

### 14.2 Upload Single File  

Upload hello.txt:  

    docker run --rm \  
      -v /data/rustfs/mc-config:/root/.mc \  
      -v /tmp/rustfs-test:/test \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      cp /test/hello.txt rustfs/app-uploads/hello.txt  

Check:  

    docker run --rm \  
      -v /data/rustfs/mc-config:/root/.mc \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      ls rustfs/app-uploads  

---

### 14.3 Upload Directory  

Upload logs directory:  

    docker run --rm \  
      -v /data/rustfs/mc-config:/root/.mc \  
      -v /tmp/rustfs-test:/test \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      cp --recursive /test/logs rustfs/app-uploads/  

Check:  

    docker run --rm \  
      -v /data/rustfs/mc-config:/root/.mc \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      ls --recursive rustfs/app-uploads  

---

### 14.4 Upload Large File  

Upload 100Mi file:  

    docker run --rm \  
      -v /data/rustfs/mc-config:/root/.mc \  
      -v /tmp/rustfs-test:/test \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      cp /test/file-100m.bin rustfs/app-uploads/file-100m.bin  

Check object:  

    docker run --rm \  
      -v /data/rustfs/mc-config:/root/.mc \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      stat rustfs/app-uploads/file-100m.bin  

---

### 14.5 Download Object  

Create download directory:  

    mkdir -p /tmp/rustfs-download  

Download hello.txt:  

    docker run --rm \  
      -v /data/rustfs/mc-config:/root/.mc \  
      -v /tmp/rustfs-download:/download \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      cp rustfs/app-uploads/hello.txt /download/hello.txt  

Download large file:  

    docker run --rm \  
      -v /data/rustfs/mc-config:/root/.mc \  
      -v /tmp/rustfs-download:/download \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      cp rustfs/app-uploads/file-100m.bin /download/file-100m.bin  

Check:  

    ls -lh /tmp/rustfs-download  

---

### 14.6 Verify File Consistency  

Execute:  

    sha256sum /tmp/rustfs-test/hello.txt /tmp/rustfs-download/hello.txt  
    sha256sum /tmp/rustfs-test/file-100m.bin /tmp/rustfs-download/file-100m.bin  

If the hashes match, it indicates the uploaded and downloaded object contents are consistent.  

---

## 15. Data Persistence Verification After Container Restart

### 15.1 Restarting the Container

Run:

    docker restart rustfs-single

Check:

    docker ps | grep rustfs-single
    docker logs rustfs-single --tail=100

---

### 15.2 Checking the API

Run:

    curl -i http://10.0.0.51:9000/health

Expected:

    HTTP/1.1 200 OK

---

### 15.3 Rechecking Bucket and Object

Run:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs

Check objects:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs/app-uploads

Expected:

    app-uploads still exists.
    hello.txt still exists.
    logs/app.log still exists.
    file-100m.bin still exists.

---

### 15.4 Conclusion

If data still exists after container restart, it indicates:

    Data is not only stored in the container's temporary layer.
    Data has been persisted to the host's /data/rustfs.
    -v /data/rustfs:/data mount is effective.

Note:

    This only proves data retention during container restart.
    Does not imply data retention in case of disk failure.
    Does not imply node failure recovery.
    Does not imply production high availability.
    Multi-node cluster is required to verify node failure scenarios.

---

## Sixteen, Observing the Host Data Directory

### 16.1 Checking Data Directory Capacity

Run:

    du -sh /data/rustfs
    find /data/rustfs -maxdepth 3 -type d | head -50
    find /data/rustfs -maxdepth 3 -type f | head -50

Notes:

    You can observe the directory structure.
    Do not manually modify internal files.
    Do not manually delete object storage internal data files.

---

### 16.2 High-Risk Warning

Prohibited actions:

    rm -rf /data/rustfs/*
    mv /data/rustfs /data/rustfs.bak
    chmod -R 777 /data/rustfs
    Manual modification of internal object data in /data/rustfs
    Manual deletion of internal metadata

Object deletion should be done through:

    mc rm
    AWS CLI
    RustFS Console
    S3 API

---

## Seventeen, AWS CLI Access Verification

### 17.1 Installing AWS CLI

Ubuntu example:

    apt update
    apt install -y awscli

Check:

    aws --version

If the apt version is outdated, it does not affect basic experiments.

---

### 17.2 Configuring Temporary Environment Variables

Run:

    export AWS_ACCESS_KEY_ID="rustfsadmin"
    export AWS_SECRET_ACCESS_KEY="RustFSAdmin@123456"
    export AWS_DEFAULT_REGION="us-east-1"

---

### 17.3 Checking Bucket

Run:

    aws --endpoint-url http://10.0.0.51:9000 s3 ls

Expected to see:

    app-uploads

---

### 17.4 Uploading Objects

Run:

    echo "upload by aws cli" > /tmp/aws-cli-test.txt

    aws --endpoint-url http://10.0.0.51:9000 \
      s3 cp /tmp/aws-cli-test.txt s3://app-uploads/aws-cli-test.txt

Check:

    aws --endpoint-url http://10.0.0.51:9000 \
      s3 ls s3://app-uploads/

---

### 17.5 Downloading Objects

Run:

    aws --endpoint-url http://10.0.0.51:9000 \
      s3 cp s3://app-uploads/aws-cli-test.txt /tmp/aws-cli-test-download.txt

Check:

    cat /tmp/aws-cli-test-download.txt

---

### 17.6 Common AWS CLI Issues

If you encounter:

    SignatureDoesNotMatch

Check:

    Whether the AccessKey is correct.
    Whether the SecretKey is correct.
    Whether the Region matches.
    Whether system time is synchronized.
    Whether S3 v4 signing is used.
    Whether the Endpoint is correct.
    Whether path-style access is required.

If you encounter:

    EndpointConnectionError

Check:

    Whether RustFS is running.
    Whether port 9000 is listening.
    Whether firewall rules allow traffic.
    Whether the Endpoint URL is correct.

If you encounter:

    AccessDenied

Check:

    Whether the account credentials are correct.
    Whether the Bucket exists.
    Whether permissions are sufficient.

---

## Eighteen, Error Key Access Verification

### 18.1 Testing with an Incorrect SecretKey

Run:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-wrong http://10.0.0.51:9000 rustfsadmin 'WrongPassword123'

Try accessing: /think

docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-wrong

Expected output:

    Authentication failed.
    AccessDenied.
    SignatureDoesNotMatch.
    Or similar authentication errors.

Notes:

    Objects cannot be accessed with an incorrect access key.
    This is a basic security validation.

---

## 19. Common Issues Troubleshooting

### 19.1 Container startup failure

Check:

    docker ps -a | grep rustfs-single
    docker logs rustfs-single --tail=200

Common causes:

    Port conflict.
    Insufficient permissions for data directory.
    Image not found.
    Incorrect startup parameters.
    SecretKey is too weak or format is invalid.
    Data directory is not writable.

Resolution:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'
    ls -ld /data/rustfs
    chown -R 10001:10001 /data/rustfs
    docker logs rustfs-single --tail=200

---

### 19.2 Permission denied

Symptoms:

    "permission denied" appears in docker logs

Troubleshooting:

    ls -ld /data/rustfs
    id
    docker inspect rustfs-single | grep -i user -C 3

Resolution:

    chown -R 10001:10001 /data/rustfs
    docker restart rustfs-single

Notes:

    The RustFS container runs as a non-root user on the host.
    The mounted directory must allow UID 10001 to write.

---

### 19.3 Port 9000 is inaccessible

Troubleshooting:

    docker ps | grep rustfs-single
    ss -lntp | grep ':9000'
    curl -i http://127.0.0.1:9000/health
    curl -i http://10.0.0.51:9000/health
    docker logs rustfs-single --tail=100

Possible causes:

    Container is not running.
    Port is not mapped.
    Firewall interception.
    RustFS is not listening properly.
    Incorrect access IP.

---

### 19.4 Console cannot log in

Troubleshooting:

    http://10.0.0.51:9001/rustfs/console
    docker logs rustfs-single --tail=100
    ss -lntp | grep ':9001'

Check:

    Is the AccessKey "rustfsadmin"?
    Is the SecretKey "RustFSAdmin@123456"?
    Are old container parameters being used?
    Were environment variables modified without rebuilding the container?
    Is the Console enabled?

If you need to modify AccessKey / SecretKey, it is recommended to:

    Backup data first.
    Stop the container.
    Clearly determine if it will affect existing accounts.
    Recreate the container.
    Do not change access keys arbitrarily in production environments.

---

### 19.5 mc alias failure

Troubleshooting:

    curl -i http://10.0.0.51:9000/health
    docker run --rm -v /data/rustfs/mc-config:/root/.mc registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z alias list
    docker logs rustfs-single --tail=100

Common causes:

    Endpoint error.
    AccessKey error.
    SecretKey error.
    Network connectivity issues.
    Port 9000 is not open.
    System time mismatch.
    S3 client compatibility issues.

---

### 19.6 Upload object failure

Troubleshooting:

    docker logs rustfs-single --tail=200
    df -hT /data/rustfs
    ls -ld /data/rustfs
    docker run --rm -v /data/rustfs/mc-config:/root/.mc registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z ls rustfs

Common causes:

    Bucket does not exist.
    Insufficient permissions.
    Data directory is not writable.
    Insufficient disk space.
    Incorrect client command path.
    Endpoint error.

---

### 19.7 Bucket loss after container restart

High probability causes:

    Incorrectly mounted host data directory.
    Previous data was written inside the container.
    Changed data directory when recreating the container.
    Deleted /data/rustfs.
    Used different data paths.

Check:

    docker inspect rustfs-single | grep -i Mounts -A20
    ls -lah /data/rustfs
    du -sh /data/rustfs

Correct startup must include:

    -v /data/rustfs:/data

---

## 20. Production Security Reminders

### 20.1 Experimental accounts cannot be used directly in production

This document uses:

    rustfsadmin
    RustFSAdmin@123456

Only suitable for experiments.

Production requirements:

    Use strong passwords.
    Use independent business accounts.
    Root / Admin accounts are only for initialization.
    AccessKey / SecretKey should not be written to public repositories.
    Keys should be rotated regularly.
    Keys can be disabled after leakage.
    Different businesses should use different Buckets and different keys.

---

### 20.2 External access must use HTTPS

This document's single-machine experiment uses:

    http://10.0.0.51:9000

Only for internal network experiments.

Production external access must:

    HTTPS
    Valid certificate
    Minimal permissions
    Source restriction
    Access logs
    Management entry isolation

Next 07 will be implemented via Nginx:

    https://s3.rustfs.local

---

### 20.3 Console cannot be exposed to the public internet

# RustFS Console is the management entry point.

Production requirements:

    Do not expose to the public internet directly.
    Only allow access from the operations network segment.
    Recommend accessing via VPN / bastion host.
    Use HTTPS.
    Use strong authentication.
    Retain access logs.

---

## 21. Single-machine mode boundaries

Single-machine mode can verify:

    Service startup
    Port listening
    Console access
    S3 API basic access
    Bucket creation
    Object upload/download
    Data directory persistence
    Container restart recovery

Single-machine mode cannot verify:

    Node high availability
    Disk failure recovery
    Multi-node data distribution
    Erasure Coding production reliability
    Unified entry load balancing
    Multi-node failure recovery
    Production performance
    Production upgrade path

Conclusion:

    Single-machine Docker is for initial experimentation.
    It is not a production architecture.
    Subsequent deployment must transition to multi-node cluster.

---

## 22. Experiment cleanup

### 22.1 Stop container

Execute:

    docker stop rustfs-single

---

### 22.2 Start container

If temporarily stopped and needs to resume:

    docker start rustfs-single

---

### 22.3 Remove container

High-risk warning:

    Removing the container will not automatically delete /data/rustfs.
    However, if data exists only inside the container, deleting the container will result in data loss.
    This document uses a host-mounted directory, so data remains in /data/rustfs.

Remove:

    docker rm -f rustfs-single

---

### 22.4 Remove data directory

High-risk warning:

    The following command will delete RustFS single-machine experiment data.
    Do not execute in production environments.

Execute after confirming no data is needed:

    rm -rf /data/rustfs

---

### 22.5 Remove test files

Execute:

    rm -rf /tmp/rustfs-test
    rm -rf /tmp/rustfs-download
    rm -f /tmp/aws-cli-test.txt
    rm -f /tmp/aws-cli-test-download.txt

---

### 22.6 Remove mc configuration

If only cleaning up experimental alias:

    rm -rf /data/rustfs/mc-config

If continuing RustFS experiments, it is recommended to retain this.

---

## 23. Completion criteria for this document's hands-on practice

After completing this document, the following should be at least achieved:

| Item | Standard |
|---|---|
| Image version | Use rustfs/rustfs:1.0.0-alpha.99 fixed version |
| Image synchronization | Already synchronized to registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99 |
| Data directory | /data/rustfs has been created |
| Directory permissions | UID 10001 has write access |
| Container | rustfs-single Running |
| API port | 10.0.0.51:9000 is accessible |
| Console port | 10.0.0.51:9001 is accessible |
| Health check | /health returns 200 |
| mc alias | rustfs alias configuration is successful |
| Bucket | app-uploads creation is successful |
| Object upload | hello.txt, logs, file-100m.bin upload is successful |
| Object download | Download is successful |
| Data verification | sha256sum matches |
| Restart verification | After docker restart, Bucket and Object still exist |
| Troubleshooting capability | Can handle port, permission, authentication, and access failure issues |

---

## 24. Interview response thinking

If asked in an interview:

    How to deploy RustFS single-machine Docker? What will you verify?

You can respond:

    RustFS single-machine Docker deployment belongs to SNSD, which is single-node single-disk mode, mainly used for learning and basic functionality verification, unsuitable for direct production high availability architecture.
    I will first fix the image version, such as rustfs/rustfs:1.0.0-alpha.99, instead of using latest. In domestic network environments, I will first pull from the official image and then tag it to my own Alibaba Cloud image repository, such as registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99, and use my own image repository for subsequent deployments to ensure experiment reproducibility.
    Before deployment, I will check Docker, ports, disks, and time synchronization. RustFS container's default API port is 9000, and the commonly used Console port is 9001. I will place the data directory in the host's /data/rustfs and mount it into the container via -v /data/rustfs:/data to avoid data existing only in the container's temporary layer. Since the UID in the RustFS container is usually 10001, the host directory needs to be chown -R 10001:10001 /data/rustfs, otherwise permission denied may occur.
    After startup, I will check docker ps, docker logs, and ss -lntp, and access http://10.0.0.51:9000/health to verify API health status, then access http://10.0.0.51:9001/rustfs/console to verify Console.
    For client verification, I will configure mc alias, create a Bucket, such as app-uploads, then upload hello.txt, small directory, and 100Mi test file, download them back for sha256sum verification. Finally, restart the container and use mc ls again to check if the Bucket and Object still exist, verifying that data is indeed persisted to the host directory.
    For troubleshooting, if the container fails to start, first check docker logs; if port access fails, check ss -lntp, firewall, and curl /health; if upload fails, check Bucket, permissions, data directory, and disk space; if Console login fails, check AccessKey, SecretKey, and whether Console is enabled.
    In production, I would not directly use single-machine mode to carry critical business. Single-machine mode can only prove that RustFS basic functionality is available. Subsequent steps would involve multi-node clusters, unified HTTPS entry, permission governance, monitoring alerts, failure drills, and performance stress testing.

---

## 25. Summary of this document

This document completed the RustFS single-machine deployment practice:

1. RustFS single-node mode belongs to SNSD.
2. SNSD is suitable for learning and basic validation, but not suitable for production high availability.
3. This article uses rustfs/rustfs:1.0.0-alpha.99 as a fixed version.
4. It is not recommended to use latest as the official experimental version.
5. For domestic network environments, it is recommended to synchronize images to Alibaba Cloud repository or Harbor.
6. The target image in this article is registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99.
7. RustFS API port uses 9000.
8. RustFS Console port uses 9001.
9. RustFS data directory is planned as /data/rustfs.
10. The data directory inside the container is /data.
11. Must persist data through -v /data/rustfs:/data.
12. The UID of the RustFS container runtime user is usually 10001.
13. The host data directory needs to be chown to 10001.
14. /health can be used for API health check.
15. Console access path is /rustfs/console.
16. mc can be used to configure alias, create Bucket, upload and download objects.
17. After upload/download, use sha256sum to verify file consistency.
18. Buckets and Objects still exist after container restart, indicating the host data directory mount is effective.
19. Single-node mode cannot prove node high availability and disk failure recovery capability.
20. The next article will enter RustFS cluster deployment practice: multi-node, multi-disk and access entry.

---

## Twenty-six, Reference Documents

RustFS official website:

    https://rustfs.com/

RustFS official documentation:

    https://docs.rustfs.com/

RustFS Docker installation documentation:

    https://docs.rustfs.com/installation/docker/

RustFS Docker Hub:

    https://hub.docker.com/r/rustfs/rustfs

RustFS GitHub:

    https://github.com/rustfs/rustfs

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

AWS CLI S3 documentation:

    https://docs.aws.amazon.com/cli/latest/reference/s3/

AWS S3 API documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

Docker official documentation:

    https://docs.docker.com/

Nginx official documentation:

    https://nginx.org/en/docs/