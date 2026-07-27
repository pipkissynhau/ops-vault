# MinIO Deployment Modes: Standalone, Single-Node Multiple Disks, and Multi-Node Clusters

Recommended Path: 05-Storage/02-MinIO/02-MinIO Deployment Modes: Standalone, Single-Node Multiple Disks, and Multi-Node Clusters.md

Tags: #MinIO #DeploymentModes #StandaloneDeployment #Single-NodeMultipleDisks #Multi-NodeClusters #Docker #ErasureCoding #ObjectStorage #S3 #AdvancedSRE #ProductionOps

---

## I. Document Introduction

This article is the second part of the MinIO module, focusing on several common deployment modes of MinIO.

The previous article covered:

- Basic concepts of object storage
- Bucket / Object / Prefix
- S3 API
- AccessKey / SecretKey
- Basic operations of the mc client
- Minimum standalone Docker experiment
- Basic understanding of Erasure Coding

This article continues to explore the deployment architecture, addressing key questions such as:

    What are the deployment modes of MinIO?
    In what scenarios is a single-machine single-disk configuration suitable?
    What are the differences between a single-node multiple-disks configuration and a single-machine single-disk configuration?
    Why is a multi-node multiple-disks configuration closer to production use cases?
    How should nodes, disks, and ports be planned for MinIO?
    Why is it not recommended to deploy MinIO in a standalone mode in a production environment?
    Why must internal node communications use HTTP while external access uses HTTPS?
    Which deployment modes are only suitable for experiments and which ones are suitable for production?

This article still emphasizes practicality and will provide direct executable Docker commands.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the three deployment modes of MinIO: single-machine single-disk, single-node multiple-disks, and multi-node multiple-disks.
2. Use Docker to start a single-machine single-disk MinIO instance.
3. Use Docker to start a single-node multiple-disks MinIO instance.
4. Understand that a single-node multiple-disks configuration can help verify Erasure Coding but does not provide node-level high availability.
5. Recognize that a multi-node multiple-disks configuration is a more production-ready approach.
6. Plan the nodes, ports, and data directories for a 4-node MinIO distributed cluster.
7. Understand the difference between MinIO API port 9000 and Console port 9001.
8. Comprehend the architectural boundaries between internal HTTP communications and external HTTPS access.
9. Analyze the potential failure risks of different deployment modes.
10. Prepare for the next article on "4-Node Multi-Disk Distributed Cluster Deployment Practice".

---

## III. Experimental Environment

### 3.1 Experimental IP Range

This module uses the following IP range for experiments:

    10.0.0.0/24

Addresses that should be avoided include:

| IP | Purpose |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |

---

### 3.2 MinIO Node Planning

This article covers three experimental modes:

| Mode | Nodes Used | Purpose |
|---|---|---|
| Single-machine single-disk | 10.0.0.41 | Minimum setup, understanding API and Console |
| Single-node multiple-disks | 10.0.0.41 | Understanding Erasure Coding and multiple data directories |
| Multi-node multiple-disks | 10.0.0.41-10.0.0.44 | Understanding distributed cluster architecture |

Node planning:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | Standalone / Distributed Node 1 |
| 10.0.0.42 | minio-node02 | Distributed Node 2 |
| 10.0.0.43 | minio-node03 | Distributed Node 3 |
| 10.0.0.44 | minio-node04 | Distributed Node 4 |
| 10.0.0.45 | minio-client | mc client, optional |
| 10.0.0.46 | minio-entry | Nginx / HTTPS unified entry, optional |

---

### 3.3 Operating System

Default:

    Ubuntu Server 22.04.5 LTS

Alternative:

    Rocky Linux 9

This article uses Ubuntu Server 22.04.5 LTS as the default.

---

### 3.4 ImageThe mc can be connected.
Bucket can be created.
Objects can be uploaded and downloaded.

---

### 7.2 Creating a Data Directory

Execute on minio-node01:

    mkdir -p /data/minio-single/data

Check:

    ls -ld /data/minio-single/data

---

### 7.3 Starting a Single-Disk MinIO Instance

Execute:

    docker run -d \
      --name minio-single \
      --restart unless-stopped \
      -p 9000:9000 \
      -p 9001:9001 \
      -e MINIO_ROOT_USER=minioadmin \
      -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \
      -v /data/minio-single/data:/data \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
      server /data --console-address ":9001"

---

### 7.4 Checking the Container Status

    docker ps | grep minio-single

View logs:

    docker logs -f minio-single

If API and Console information are displayed, it indicates successful startup.

---

### 7.5 Checking Ports

    ss -lntp | grep -E '9000|9001'

Expected result:

    Port 9000 is listening.
    Port 9001 is listening.

---

### 7.6 Accessing the Console

Access via browser:

    http://10.0.0.41:9001

Login:

    Username: minioadmin
    Password: MinioAdmin@123456

---

### 7.7 Configuring mc Alias

Create an mc configuration directory:

    mkdir -p /data/minio/mc-config

Set the alias:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set single http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Check the alias:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 7.8 Checking Service Information

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info single

---

### 7.9 Creating a Bucket

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb single/single-demo

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls single

---

### 7.10 Uploading and Downloading Objects

Create a test file:

    mkdir -p /tmp/minio-single-demo

    echo "hello minio single mode" > /tmp/minio-single-demo/hello.txt

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-single-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/hello.txt single/single-demo/hello.txt

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls single/single-demo

Download:

    rm -f /tmp/minio-single-demo/hello-download    docker logs -f minio-multi-disk

---

### 8.6 Configuring mc Alias

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set multidisk http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

    To view service information:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info multidisk

---

### 8.7 Creating a Bucket and Uploading an Object

    Create a Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb multidisk/multidisk-demo

    Create a test file:

    mkdir -p /tmp/minio-multidisk-demo

    dd if=/dev/zero of=/tmp/minio-multidisk-demo/file-50m.bin bs=1M count=50

    Upload the file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-multidisk-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/file-50m.bin multidisk/multidisk-demo/file-50m.bin

    To check the file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls multidisk/multidisk-demo

---

### 8.8 Viewing the Distribution of Data Directories

    On the host machine:

    find /data/minio-multi-disk -maxdepth 4 -type f | head -100

    To view directory sizes:

    du -sh /data/minio-multi-disk/disk*

    Note:

    You can see that MinIO stores data and metadata in multiple directories.
    Do not manually modify these directories.
    Do not delete files directly from the data directories.
    Object operations must be performed through the S3 API or mc.

---

### 8.9 Verifying Download

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-multidisk-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp multidisk/multidisk-demo/file-50m.bin /demo/file-50m-download.bin

    To check the downloaded file:

    ls -lh /tmp/minio-multidisk-demo/file-50m-download.bin

---

### 8.10 Risks of Multiple Disks on a Single Node

    Using multiple disks on a single node can help with some disk failure scenarios, but it does not address the issue of node failures.

    Risks:

    If a node crashes, the entire service becomes unavailable.
    If the operating system is damaged, the service stops working.
    If there are Docker issues, the service may fail.
    A network interruption can also cause service disruption.
    If multiple directories are on the same physical disk, a disk failure could lead to data loss.
    Single-node setups are not suitable for high-availability production environments.

    For production use:

    Single-node with multiple disks is a transitional approach for learning about Erasure Coding.
    In production, multi-node systems with multiple disks are recommended.

---

## Chapter 9: Experimental Plan for Multi-Node, Multiple-Disk Deployment

### 9.1 Explanation of This Section

    This section only provides a plan for the multi-node, multiple-disk deployment model; it does not include all the detailed deployment commands.

    A complete guide to deploying a multi-node, multiple-disk distributed system will be provided in the next article:

    03-MinIO DistributedMore suitable for front-end Nginx / Load Balancer setup
Easier to perform monitoring and fault tolerance testing

However, it also brings the following challenges:

Increased complexity in node planning
Greater network dependencies
Higher requirements for disk configuration
More critical failure recovery processes
Essential need for comprehensive monitoring and alerts
Necessary planning for backup and migration strategies

---

## Section 10: Internal HTTP vs. External HTTPS Policies

### 10.1 Unified Policy for This Module

The MinIO module will uniformly adopt the following approach:

- Use HTTP for internal node communication.
- Use HTTPS for external client access.
- Use Nginx / Load Balancer as a unified entry point.
- The backend proxy will use the MinIO API on port 9000.
- The Console on port 9001 should only be accessible by the operations team or through a restricted independent entrance.

---

### 10.2 Why Use HTTP Internally

Internal HTTP is suitable under the following conditions:

- Nodes are located within a trusted internal network.
- The network does not expose any public interfaces.
- There are firewalls or security groups in place for control.
- Communication between MinIO nodes does not cross untrusted networks.
- The operations team can control the sources of access.

Advantages include:

- Simple configuration.
- Reduced complexity in managing TLS certificates.
- Lower overhead on internal data encryption and decryption.
- Facilitates testing and troubleshooting.

---

### 10.3 Why Must External Access Use HTTPS

External access must use HTTPS for the following reasons:

- S3 requests require AccessKey authentication.
- The data being uploaded or downloaded are business-related objects.
- External networks cannot be fully trusted.
- HTTP in plaintext poses risks of data interception and tampering.
- Production security and compliance regulations typically require HTTPS.

Recommended external entry points:

- https://s3.example.com

For backend access:

- http://minio-node01:9000
- http://minio-node02:9000
- http://minio-node03:9000
- http://minio-node04:9000

---

### 10.4 Should the Console Be Open to the Public?

It is not recommended to directly open the Console to the public network.

Suggestions include:

- Only allow access from the operations network segment.
- Provide access via VPN or bastion hosts.
- Use a dedicated domain name.
- Employ HTTPS for secure communication.
- Set strong passwords.
- Do not grant root user privileges to business users.

Example configuration:

- https://minio-console.example.com

Access restrictions:

- Only allow access from the operations network segment.

---

## Section 11: Recommendations for Deployment Modes

### 11.1 Learning Phase

Recommended setup:

- Single machine with a single data disk

Objective:

- To understand APIs, the Console, the mc command-line tool, Buckets, and Objects.

---

### 11.2 Understanding Erasure Coding Phase

Recommended setup:

- Single machine with multiple data disks

Objective:

- To understand how MinIO organizes data in multiple directories.
- To comprehend the basic principles of Erasure Coding.

Note:

- If multiple directories are stored on the same physical disk, this setup is only suitable for learning purposes.

---

### 11.3 Approaching Production Phase

Recommended setup:

- A distributed cluster with 4 nodes and multiple data disks

Objective:

- To understand real-world distributed deployment scenarios.
- To test for node failures and disk failures.
- To verify the functionality of Nginx as a unified entry point.
- To test the management capabilities of the mc command-line tool.
- To ensure proper monitoring and alert mechanisms are in place.
- To validate backup and migration procedures.

---

### 11.4 Production Phase

Production recommendations include:

- At least multiple nodes with multiple data disks.
- Use stable versions of software.
- Employ official domain names and HTTPS for secure connections.
- Clearly define the access boundaries between APIs and the Console.
- Implement minimum permission policies for business users.
- Utilize Prometheus for monitoring.
- Set up alerts for capacity and node status.
- Perform regular backups and recovery tests.
- Establish well-defined processes for handling changes and failures.

---

## Section 12: Pre-Deployment Checklist

### 12.1 Operating System Checks

Verify the following:

- Hostname
- IP address
- Route settings
- Time synchronization
- Disk space usage
- List of available disks

Requirements:

- The hostname should be clear and unique.
- The IP address should be fixed.
- Time synchronization should be accurate.
- Data disks should be properly mounted.
- The system disk should have sufficient capacity.

---

### 12.2 Docker Checks

Verify the following:

- Docker version
- Basic Docker information
- List of available MinIO images

If Docker is not installed, install it first.

---

### 12.3---

### 15.3 Clearing mc Configuration

If you are certain that you will no longer use it:

    rm -rf /data/minio/mc-config

High-risk Warning:

    Deleting the data directory will erase all object data.
    This operation should only be performed in a testing environment.
    In a production environment, directly deleting the data directory is strictly prohibited.

---

## Sixteen. Interview Answer Guidelines

If you are asked during an interview:

    What are the deployment modes of MinIO? How should it be deployed for production use?

You can answer as follows:

    Common deployment modes of MinIO include single-machine single-disc, single-node multiple-disks, and multi-node multiple-disks.
    The single-machine single-disc configuration is the simplest and suitable for learning, development, and functional testing, but it lacks high availability. A failure in either the node or the disk can lead to service interruptions or data loss.
    Single-node multiple-disks allow MinIO to use multiple data directories or disks, which helps in understanding Erasure Coding and provides some tolerance to disk failures. However, this configuration still cannot handle node failures and is therefore not recommended for production scenarios requiring high availability.
    For production environments, it is more advisable to deploy MinIO in a multi-node multiple-disks setup, such as 4 nodes, each with multiple data disks. This approach enables Erasure Coding for both disk and node-level fault tolerance, and an external HTTPS gateway like Nginx or LB can be used to provide a unified access point.
    In production settings, I would ensure that MinIO nodes operate within a trusted internal network, using HTTP for internal communications to reduce the complexity and overhead of TLS management. External client access must be secured via HTTPS to protect S3 credentials and object data. Additionally, it is not recommended to expose the Console directly over the public internet, and business operations should avoid using the root user instead of assigning independent AccessKeys with minimal permission policies.
    In addition to deployment, it is essential to monitor nodes, disks, bucket capacity, the number of objects, and API error rates. Regular backups and migrations should be performed using tools like mc mirror or cross-cluster solutions. It should be noted that Erasure Coding cannot replace proper data backup.

---

## Seventeen. Summary of This Article

This article has covered various aspects of MinIO deployment:

1. Common deployment modes include single-machine single-disc, single-node multiple-disks, and multi-node multiple-disks.
2. Single-machine single-disc is suitable for learning and testing but not for production use.
3. Single-node multiple-disks help in understanding Erasure Coding but cannot handle node failures.
4. Multi-node multiple-disks are a more appropriate choice for production environments.
5. The S3 API port is 9000, and the Web Console port is 9001.
6. The mc client should connect to the 9000 API port, not the 9001 Console port.
7. External access to production systems must use HTTPS for security.
8. Internal node communications can use HTTP within a trusted network.
9. It is not recommended to expose the Console directly over the public internet.
10. Business operations should avoid using the root user.
11. Data directories should not be deleted or modified arbitrarily.
12. In multi-node setups, all nodes must use identical startup parameters.
13. Successful deployment depends on hostname resolution, network connectivity, time synchronization, and proper data directory configuration.
14. Erasure Coding in MinIO does not replace traditional data backup methods.
15. The next article will focus on the practical aspects of deploying a 4-node multi-disc MinIO cluster.

---

## Eighteen. References

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Docker Deployment Documentation:

    https://min.io/docs/minio/container/index.html

MinIO Distributed Deployment Documentation:

    https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html

MinIO Erasure Coding Documentation:

    https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO User and Permission Management Documentation:

    https://min.io/docs/minio/linux/administration/identity-access-management.html

AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html