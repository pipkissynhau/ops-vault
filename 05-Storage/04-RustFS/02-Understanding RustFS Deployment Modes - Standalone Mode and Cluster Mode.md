# Understanding RustFS Deployment Modes: Standalone Mode and Cluster Mode

Recommended Path: 05-Storage/04-RustFS/02-RustFS Deployment Modes: Standalone Mode and Cluster Mode.md

Tags: #RustFS #Object Storage #S3 #Deployment Modes #Standalone Deployment #Cluster Deployment #Docker #Multiple Nodes #Multiple Disks #Reverse Proxy #HTTPS #Advanced SRE #Production Operations

---

## I. Document Introduction

This article is the second part of the RustFS module, focusing on understanding RustFS's deployment modes, node planning, disk planning, access entry design, and production considerations.

The previous article covers:

    RustFS Basics: S3-Compatible Object Storage and Use Cases

This article does not delve directly into complete deployment commands. A detailed standalone deployment will be discussed in Article 03, while multi-node cluster deployments will be covered in Article 04.

This article addresses the following key questions:

    What are the deployment modes of RustFS?
    What is suitable for the standalone mode?
    What is suitable for the single-node multiple-disks mode?
    What is suitable for the multi-node multiple-disks cluster mode?
    Why do object storage clusters typically require multiple nodes and disks?
    How should RustFS nodes, disks, ports, domain names, and reverse proxies be planned?
    How should internal HTTP and external HTTPS be designed?
    What are the differences between Docker deployment and VM/naked machine deployment?
    Why can't a standalone experiment be directly used in production?
    Why must RustFS's version, compatibility, fault recovery, and data reliability be verified before production?

This article emphasizes one principle:

    The RustFS module should follow this sequence: "First, verify in a standalone environment; then move on to multi-node clusters; next, unify the access entry; finally, ensure security and operations."

    Standalone deployment is used for learning and functional verification.
    The single-node multiple-disks mode is closer to the form of a production object storage cluster.
    Compatibility testing, load testing, backup, recovery, and fault drills must be completed before production use.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the role of the RustFS standalone mode.
2. Understand the role of the single-node multiple-disks mode.
3. Understand the role of the multi-node multiple-disks mode.
4. Comprehend deployment patterns such as SNSD, SNMD, and MNMD.
5. Recognize why distributed object storage cannot be assessed solely based on whether containers are started.
6. Plan a RustFS standalone Docker experimental environment.
7. Plan a RustFS multi-node Docker/VM cluster environment.
8. Plan RustFS data directory and disk mounting paths.
9. Plan RustFS API ports, management ports, and unified access entries.
10. Understand the differences between internal HTTP and external HTTPS.
11. Comprehend the role of Nginx/LB reverse proxies in front of RustFS.
12. Determine whether a scenario is suitable for a standalone, test cluster, or production cluster.
13. Recognize similarities in deployment methods between RustFS and MinIO.
14. Understand that new RustFS solutions need to be carefully verified before production.
15. Lay the foundation for subsequent standalone deployment in Article 03 and multi-node cluster deployment in Article 04.

---

## III. Overview of RustFS Deployment Modes

RustFS can be categorized into three deployment modes based on the number of nodes and disks:

    Single Node Single Disk Mode
    Single Node Multiple Disks Mode
    Multiple Nodes Multiple Disks Mode

These can also be represented as:

    SNSD: Single Node Single Disk
    SNMD: Single Node Multiple Disk
    MNMD: Multiple Node Multiple Disk

In Chinese, this is interpreted as:

| Mode | Chinese Name | Description |
|---|---|---|
| SNSD | 单节点单磁盘 | One node, one data directory or one data disk |
| SNMD | 单节点多磁盘 | One node, multiple data directories or multiple data disks |
| MNMD | 多节点多磁盘 | Multiple servers, each with at least one or more data disks |

The recommended learning order is:

    1. First, learn SNSD to understand service startup, ports, data directories, and S3 access.
    2. Then, learn SNMD to understand single-node multiple-disks and capacity planning.
    3. Finally, learn MNMD to comprehend distributed object storage, node failures, and unified access.

---

## IV. Single Node Single Disk Mode: SNSD

### 4.1 Mode Definition

SNSD stands for:

    Single Node Single Disk

That is:

    One RustFS node
    One data directory
    One Docker container or process
    One S3 API entry point

Example:

    rustfs-node01
#### Verify Disk Directory Permissions
#### Verify Multi-Disk Boot Parameters
#### Verify Object Writing Distribution
#### Small-Scale Test Environment

---

### 5.4 SNMD Inappropriate Scenarios

Not suitable for:

- High availability in production environments
- Tolerance to node failures
- Cross-node recovery
- Multi-node disaster recovery
- Core business object storage

Reasons:

- Although there are multiple disks, there is still only one node.
- After a server crash, the entire object storage service becomes unavailable.
- Node-level failures cannot be compensated for by other nodes.

---

### 5.5 Key Points to Learn About SNMD

Key areas of focus in SNMD:

- Planning for multiple data directories
- Mounting multiple disks
- Using independent file systems for each disk
- Setting permissions for data directories
- Ensuring consistency in disk capacity and performance
- Configuring container startup parameters
- Understanding the capacity expansion limits of a single node

---

## VI. Multi-Node, Multi-Disk Mode: MNMD

### 6.1 Mode Definition

MNMD stands for:

- Multiple Node Multiple Disk

That is:

- Multiple RustFS nodes
- Each node has at least one or more data directories
- Multiple nodes form a distributed object storage cluster
- Clients access through a unified interface

Example configuration:

| IP | Host Name | Data Directory |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | /data/rustfs |
| 10.0.0.52 | rustfs-node02 | /data/rustfs |
| 10.0.0.53 | rustfs-node03 | /data/rustfs |
| 10.0.0.54 | rustfs-node04 | /data/rustfs |

Unified access interface:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.56 | rustfs-entry | Nginx / Load Balancer for HTTPS access |

Clients:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.55 | rustfs-client | Used for mc, AWS CLI, and application testing |

---

### 6.2 MNMD Topology

    ┌─────────────────────────────────────┐
    │ rustfs-node01                       │
    │ 10.0.0.51                           │
    │ /data/rustfs                        │
    └───────────────────┬─────────────────┘
                        │
    ┌───────────────────┼─────────────────┐
    │                   │                 │
    v                   v                 v
    ┌─────────────────────────────────────┐
    │ rustfs-node02                       │
    │ 10.0.0.52                           │
    │ /data/rustfs                        │
    └─────────────────────────────────────┘

    ┌─────────────────────────────────────┐
    │ rustfs-node03                       │
    │ 10.0.0.53                           │
    │ /data/rustfs                        │
    └─────────────────────────────────────┘

    ┌─────────────────────────────────────┐
    │ rustfs-node04                       │
    │ 10.0.0.54                           │
    │ /data/rustfs                        │
    └─────────────────────────────────────┘

                    ^
                    |
                    | Internal HTTP / Cluster Communication |
                    |
    ┌─────────────────────────────────────┐
    │ rustfs-entry                        │
    │ 10.0.0.56                           │
    │ Nginx / Load Balancer                          │
    │ HTTPS: 443                          │
    └─────────────────────────────────────┘
                    ^
                    |
                    | External HTTPS S3 API |
                    |
    ┌─────────────────────────────────────┐
    │ rustfs-client                       │
    │ 10.0.0.55                           │
    │ mc / aws cli / app                  │
    └─────────────────────────────────────┘

---

### 6.3 MNMD Suitable Scenarios

Suitable for:

- Learning about distributed object storage
- Verifying high availability of nodes
- Conducting multi-node failure drills
- Planning multiple disks for capacity allocation
- Designing unified access interfaces
- Testing reverse proxies
- Testing HTTPS access
- Checking S3 client compatibility
- Piloting non-core business applications
- Pre-production validation environments

---

### 6.4 MNMD's Practical Value in Production

MNMD is closer to the actual production environment of object storage systems.

It can help verify🔤rustfs-entry

View:

hostname

Example settings:

hostnamectl set-hostname rustfs-node01

Example for hosts file:

cat >> /etc/hosts <<'EOF'
10.0.0.51 rustfs-node01
10.0.0.52 rustfs-node02
10.0.0.53 rustfs-node03
10.0.0.54 rustfs-node04
10.0.0.55 rustfs-client
10.0.0.56 rustfs-entry
EOF

Note:

In a test environment, you can use /etc/hosts. For a production environment, it is recommended to use an internal DNS server. Avoid relying on unstable resolution for communication between nodes.

---

### 8.3 Operating System Planning

Default system:

Ubuntu Server 22.04.5 LTS

Optional alternative:

Rocky Linux 9

Ubuntu is suitable for:

Quick experiments
Docker deployment
Nginx reverse proxy setup
To maintain consistency with previous experiments using K8s, MinIO, and Longhorn.

Rocky Linux 9 is suitable for:

Simulating a production environment similar to RHEL
Practicing firewalld configuration
Understanding SELinux settings
Additional support for enterprise-level production operations.

---

## IX. Differences Between Docker Deployment and VM Deployment

### 9.1 Docker Deployment

Characteristics of Docker deployment:

Fast startup
Easy cleanup
Facilitates maintaining a fixed image version
Suitable for single-machine experiments as well as multi-node setups
Does not affect the host machine's binary environment

Suitable for:

Learning purposes
Quick validation tests
Single-machine deployments
Multi-node Docker/VM clusters
Client compatibility testing

Points to consider:

The container data directory must be mounted on the host machine.
Data should not be stored solely within the container.
It is necessary to set a restart strategy.
Port mapping must be properly planned.
Check log and data directory permissions.
Ensure the use of a fixed image version.

---

### 9.2 VM/Bare Metal Binary Deployment

Characteristics of VM/bare metal deployment:

Closer to traditional production service setups
Processes are managed by systemd
Easy integration with system services, disks, and logging mechanisms
Suitable for production-scale evaluation

Suitable for:

Pre-production validation
System-level operations learning
Systemd management practices
Standardizing log paths
Integration with enterprise host baselines

Points to consider:

Management of binary version control
Configuration of systemd unit files
Handling environment variable settings
Ensuring proper data directory permissions
Implementing upgrade and rollback processes
Managing log segmentation
Setting up automatic service restarts

---

### 9.3 Choice for This Module

By default, this module adopts:

Single-machine Docker setup
Multi-node Docker/VM cluster configuration

Reasons:

Fast implementation.
Easy reproducibility.
Simple cleanup process.
Consistency with the experimental approach used in the MinIO module.
Suitable for learning object storage deployment patterns.
Does not require the use of Kubernetes.
Does not modify containerd.
Does not affect existing K8s clusters.

Production considerations:

A successful Docker experiment does not necessarily guarantee production readiness. In a production environment, additional considerations such as binary deployment methods, container management strategies, system services, logging, monitoring, security measures, and upgrade plans are necessary.

---

## X. Data Directory and Disk Planning

### 10.1 Data Directory for Single-Machine Mode

For single-machine mode:

/data/rustfs

Creation:

mkdir -p /data/rustfs

Verification:

df -hT /data/rustfs
lsblk -f
ls -ld /data/rustfs

---

### 10.2 Multiple Disks for a Single Node

For single-node setups with multiple disks:

/data/rustfs/disk1
/data/rustfs/disk2
/data/rustfs/disk3
/data/rustfs/disk4

Creation:

mkdir -p /data/rustfs/disk{1..4}

Verification:

df -hT /data/rustfs/disk1
df -hT /data/rustfs/disk2
df -hT /data/rustfs/disk3
df -hT /data/rustfs/disk4

---

### 10.3 Data Directory for Multiple Nodes

For multiple nodes:

/data/rustfs

or, if using multiple disks:

/data/rustfs/disk1
/data/rustfs/disk2

Recommendations:

Ensure that the path structure is consistent across all nodes.
Try to maintain similar disk capacities and performance levels among nodes.
Avoid using mixed-speed disks within the same cluster.
Do not place the data directory directly on the system disk for production use.

---

### 10.4 Recommendation for Using Separate Data Disks

For production environments:

It is recommended to use dedicated data disks.
Mount the data disks under /data             |
             | HTTPS
             v
    ┌─────────────────────────────┐
    │ Nginx / LB                  │
    │ s3.rustfs.local:443         │
    └──────────────┬──────────────┘
                   |
                   | HTTP 内网
                   v
    ┌─────────────────────────────┐
    │ rustfs-node01               │
    │ 10.0.0.51                   │
    └─────────────────────────────┘
    ┌─────────────────────────────┐
    │ rustfs-node02               │
    │ 10.0.0.52                   │
    └─────────────────────────────┘
    ┌─────────────────────────────┐
    │ rustfs-node03               │
    │ 10.0.0.53                   │
    └─────────────────────────────┘
    ┌─────────────────────────────┐
    │ rustfs-node04               │
    │ 10.0.0.54                   │
    └─────────────────────────────┘

---

## Section Thirteen: Domain Names and Certificate Planning

### 13.1 Experimental Domain Names

For experimentation, you can use:

    s3.rustfs.local
    console.rustfs.local

In the `hosts` file:

    10.0.0.56 s3.rustfs.local
    10.0.0.56 console.rustfs.local

---

### 13.2 Production Domain Names

For production, it is recommended to use:

    s3.example.com
    rustfs-console.example.com

Or, depending on the business requirements:

    s3.internal.example.com
    backup-s3.example.com

Production principles include:

- Separating API domain names from management domain names.
- Not making management domain names accessible to the public internet.
- Ensuring that API domain names use HTTPS.
- Making sure certificates are renewable and having monitoring in place for certificate expiration.
- Keeping the S3 client endpoints fixed and not changing them arbitrarily.

---

### 13.3 Certificate Planning

Sources of certificates include:

- Public CA certificates
- Enterprise internal CA certificates
- Self-signed certificates for experimentation

For production, it is recommended to use:

- Public or cross-organization trusted certificates for public access.
- Enterprise internal CA certificates for internal access.
- Self-signed certificates are only suitable for experimental purposes.
- Clients must trust the certificate chain.
- Certificate expiration can cause issues with tools like mc, AWS CLI, and SDK.

---

## Section Fourteen: Image Versioning and Domestic Network Planning

### 14.1 Principle of Using Fixed Versions

It is not recommended to use:

    latest

Reasons include:

- The `latest` version may change over time.
- Changes in the version may lead to changes in startup parameters, functionality, and the user interface.
- Such changes can make troubleshooting and reproducing experiments more difficult.

It is suggested to use fixed versions of images. Record the official source of the image, the pull time, and the tag. Then, sync these images to your own Alibaba Cloud image repository or Harbor.

---

### 14.2 Domestic Acceleration Principles

For pulling Docker images, you can use:

    mdocker acceleration addresses

However, it is more recommended to follow this process:

- Pull official images locally.
- Apply a `tag` to the local images.
- Store them in Alibaba Cloud's image repository or Harbor.
- Then, use these private repository images for subsequent operations.

Reasons include:

- The domestic network is more stable.
- This approach ensures that experiments can be reproduced consistently.
- It does not rely on real-time access to the external internet.
- It makes it easier to maintain a consistent version across different systems.
- It facilitates internal network pull operations in production environments.

---

### 14.3 Image Synchronization Template

After confirming the fixed version of RustFS, follow these steps:

    `docker pull rustfs/rustfs:<fixed_version>`
    `docker tag rustfs/rustfs:<fixed_version> \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:<fixed_version>`
    `docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:<fixed_version>`

Note:

- The specific image name and version should be determined based on subsequent actual verification.
- Do not hardcode them in advance.
- In a production environment, always refer to the official Release Notes, compatibility, and security patch assessment before selecting a version.

---

## Section Fifteen: Single-Machine Deployment Planning

### 15.1 Single-Machine Node

Node:

    10.0.0.51 rustfs-node01🔤 mc alias set rustfs https://s3.rustfs.local <ACCESS_KEY> <SECRET_KEY>

Basic Operations:

    mc mb rustfs/test-bucket
    echo "hello rustfs" > hello.txt
    mc cp hello.txt rustfs/test-bucket/
    mc ls rustfs/test-bucket
    mc cp rustfs/test-bucket/hello.txt ./hello-download.txt
    sha256sum hello.txt hello-download.txt

---

### 17.3 AWS CLI Verification

Subsequent verification commands:

    aws --endpoint-url https://s3.rustfs.local s3 ls

Create Bucket:

    aws --endpoint-url https://s3.rustfs.local s3 mb s3://test-bucket

Upload:

    aws --endpoint-url https://s3.rustfs.local s3 cp hello.txt s3://test-bucket/hello.txt

Download:

    aws --endpoint-url https://s3.rustfs.local s3 cp s3://test-bucket/hello.txt ./hello-download.txt

---

## Section Eighteen: Deployment Mode Recommendations

### 18.1 Learning Phase

Recommendation:

    SNSD Single-node Docker

Reason:

    It allows for the fastest understanding of RustFS.
    It enables quick verification of the S3 API and mc commands.
    It minimizes troubleshooting scope and is cost-effective.

---

### 18.2 Experimental Advancement Phase

Recommendation:

    MNMD Multi-node Docker / VM

Reason:

    It helps validate cluster operations, multi-node functionality, a unified interface, node failures, reverse proxies, HTTPS, and closer approximation to production environments.

---

### 18.3 Production Pilot Phase

Recommendation:

    Multi-node with multiple disks
    Independent data disks
    Unified HTTPS access
    Separate management interface
    Monitoring and alert systems
    Backup and migration strategies
    Fault recovery drills
    Fixed versions
    Gradual deployment

Non-recommendations:

    Direct production using a single-node setup.
    Using the latest version without testing.
    Production without HTTPS security.
    Lacking monitoring and recovery mechanisms.
    Deployment without compatibility tests for S3.

---

## Section Nineteen: Pre-deployment Checklist

### 19.1 General Checks

For all nodes:

    hostname
    cat /etc/os-release
    ip addr
    ip route
    timedatectl
    df -hT
    lsblk -f
    ss -lntp
    docker version
    docker info

---

### 19.2 Time Synchronization

Check:

    timedatectl

Requirement:

    System clock is synchronized: yes

Reason:

    S3 signatures and time are closely related.
    Inconsistent node times may lead to authentication failures.
    Accurate timing is essential for logging analysis.

---

### 19.3 DNS and hosts

Check:

    getent hosts rustfs-node01
    getent hosts rustfs-node02
    getent hosts rustfs-node03
    getent hosts rustfs-node04
    getent hosts s3.rustfs.local

---

### 19.4 Disks and Directories

Check:

    df -hT /data
    df -hT /data/rustfs
    ls -ld /data/rustfs
    du -sh /data/rustfs

---

### 19.5 Ports

Check:

    ss -lntp
    ss -lntp | grep 443
    ss -lntp | grep 9000
    ss -lntp | grep 9001

Note:

    The specific RustFS ports will depend on the actual configuration.
    Ensure there are no port conflicts before deployment.

---

## Section Twenty: Common Misunderstandings

### 20.1 Misunderstanding: A working single-node setup means it can be used in production.

Wrong.

A single-node setup only proves that the service can start and that basic S3 API functions are available.

It does not demonstrate:

    High availability
    Node failure recovery
    Disk failure recovery
    Data reliability
    Concurrency capabilities
    Production stability

---

### 20.2 Misunderstanding: Running Docker containers ensures data security.

Wrong.

It is necessary to confirm:

    Whether the data directory is mounted on the host machine.
    Whether it is stored on an independent data disk.
    Whether data is retained after container restarts.
    How data is recovered in case of node failures.
    The availability of backup and migration plans.

---

### 20.3 Misunderstanding: More Docker containers equal a cluster.

Wrong.

A multi-node cluster requires confirmation of:

    Whether the nodes truly form a cluster.
    Data distribution across nodes.
    Access through a unified interface.
    Expected behavior during node failures.
    Controllable recovery processes.

Simply having multiple identical single-node containers does not constitute a distributed object7. The RustFS experiment planning uses addresses 10.0.0.51-10.0.0.56.
8. Address 10.0.0.51 can be used as a stand-alone node.
9. Addresses 10.0.0.51-10.0.0.54 can be used as cluster nodes.
10. Address 10.0.0.55 is used as the client node.
11. Address 10.0.0.56 serves as the unified entry point for Nginx/Load Balancer.
12. The data directory is uniformly set to /data/rustfs.
13. For production use, it is recommended to employ independent data disks.
14. Internal node communications within a trusted network can use HTTP.
15. External client access must use HTTPS.
16. Nginx/Load Balancer can manage unified certificates, logging, source control, and backend access.
17. Docker deployment is suitable for learning purposes and quick validation.
18. VM/baremetal deployments are more closely representative of traditional production environments.
19. Image versions should be fixed; using the latest version is not recommended.
20. In domestic network environments, it is advised to synchronize images to Alibaba Cloud repositories or Harbor.
21. Before putting RustFS into production, compatibility tests, stress testing, fault drills, and recovery verifications must be completed.
22. The next article will focus on practical single-node deployment of RustFS: service startup, data directory setup, and access verification.

---

## Twenty-Five. References

RustFS Official Website:

    https://rustfs.com/

RustFS Official Documentation:

    https://docs.rustfs.com/

RustFS Docker Installation Documentation:

    https://docs.rustfs.com/installation/docker/

RustFS Linux Installation Documentation:

    https://docs.rustfs.com/installation/linux/

RustFS Multiple-Node, Multiple-Disk Installation Documentation:

    https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html

RustFS GitHub:

    https://github.com/rustfs/rustfs

RustFS Docker Hub:

    https://hub.docker.com/r/rustfs/rustfs

AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

Nginx Official Documentation:

    https://nginx.org/en/docs/

Docker Official Documentation:

    https://docs.docker.com/