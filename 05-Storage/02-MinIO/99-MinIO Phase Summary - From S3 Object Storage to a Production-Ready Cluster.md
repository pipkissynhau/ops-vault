# MinIO Phase Summary: From S3 Object Storage to a Production-Ready Cluster

Recommended Path: 05-Storage/02-MinIO/99-MinIO Phase Summary: From S3 Object Storage to a Production-Ready Cluster.md

Tags: #MinIO #Phase Summary #Object Storage #S3 #ErasureCoding #Docker #Nginx #HTTPS #Prometheus #Backup Migration #Advanced SRE #Production Operations

---

## I. Document Overview

This document serves as a phase summary for the MinIO module, providing a conclusion to the entire learning process on MinIO.

Previously, a comprehensive learning path from basic theory to production operations was completed, including:

- Basics of MinIO object storage
- S3 API, Bucket, Object, Prefix
- Single-machine Docker deployment
- Single-node multi-disc deployment
- 4-node multi-disk distributed cluster deployment
- Internal HTTP communication and external HTTPS access design
- Nginx as a unified entry point and proxy for ports 9000/9001
- mc client tool
- Bucket management and object operations
- User, Policy, AccessKey, SecretKey permission governance
- Erasure Coding for data protection
- Recovery from node and disk failures
- Prometheus monitoring, logging, and capacity management
- mc mirror for backup migration and cross-cluster synchronization

This document focuses on summarizing:

    What was learned during the MinIO phase
    The core architecture and operation logic of MinIO
    How to progress from "able to deploy" to "able to perform production operations"
    The differences between MinIO and Ceph RGW
    The relationship between MinIO and subsequent studies on RustFS and Longhorn
    What conditions a production-ready MinIO cluster should meet

---

## II. Phase Positioning

MinIO is an object storage system compatible with the S3 API.

Its core positioning is:

    To provide private object storage capabilities
    To compete with cloud providers' OSS/S3/COS solutions
    To be suitable for non-structured data such as images, attachments, backups, archives, logs, product packages, and AI datasets
    To offer access through HTTP/HTTPS APIs
    To organize data using the Bucket/Object model

From an advanced SRE perspective, learning MinIO is not just about deploying an object storage service but also about establishing a methodological approach for object storage production operations:

    How to plan nodes and disks
    How to protect object data
    How to design access entry points
    How to manage permissions and keys
    How to monitor capacity and error rates
    How to perform backup migrations
    How to handle failures
    How to determine whether a system is ready for production use

---

## III. Summary of Key Learning Areas in MinIO

### 3.1 First Mainline: Object Storage Model

Core Questions:

    What is object storage?
    What is a Bucket?
    What is an Object?
    What is a Prefix?
    What is the S3 API?

Key Takeaways:

    Understand that MinIO is neither block storage nor a traditional file system.
    Recognize that a Bucket is the top-level container in object storage.
    Know that an Object represents the actual data stored.
    Comprehend that a Prefix serves as a prefix for object keys, not a real directory.
    Understand that the S3 API is the primary way to access MinIO.
    Recognize that AccessKey/SecretKey are essential for accessing object storage.

---

### 3.2 Second Mainline: Deployment Modes

Core Questions:

    What deployment modes does MinIO offer?
    When is a single-machine single-disc setup suitable?
    When is a single-node multi-disc setup appropriate?
    Why is a multi-node multi-disk setup closer to production use?

Key Takeaways:

    Single-machine single-disc setups are ideal for learning, development, and functional testing.
    Single-node multi-disc setups help in understanding Erasure Coding but do not provide node-level high availability.
    Multi-node multi-disk setups are more suitable for production environments.
    A 4-node multi-disk cluster allows testing for node failures, disk failures, and basic high availability.
    All nodes in a MinIO distributed cluster must use the same startup endpoint list.
    Data directories must be carefully planned to avoid accidental writes to system disks.

---

### 3.3 Third Mainline: Access Entry Points

Core Questions:

    Why can't HTTP port 9000 be exposed directly?
    Why is internal communication allowed over HTTP?
    Why must external access use HTTPS?
    Should APIs and the Console be separated?
    What role does Nginx play in the architecture?

Key Takeaways:

    Port 9000 is used for the MinIO S3 API.
    Port 9001 is used for the MinIO Web Console.
    Internal node communication can use HTTP over a trusted network.
    External client access must use HTTPS.
   | 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |

MinIO Experiment Addresses:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc Client / Test Client |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entrance |

---

### 4.3 Image Version Fixation

MinIO Server Fixed Version:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Source Image:

    minio/minio:RELEASE.2025-04-22T22-12-26Z

mc Client Fixed Version:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Source Image:

    minio/mc:RELEASE.2025-04-16T18-13-26Z

Reasons for Choosing Fixed Versions:

    To avoid inconsistencies in Web Console, command parameters, log formats, and experiment results caused by changes to the latest version.
    To ensure reproducibility of experiments involving standalone Docker, distributed Docker, Nginx proxies, permissions, monitoring, backup, and migration.
    To minimize differences in capabilities between the client and the MinIO server due to similar version numbers.
    The choice of production environment versions should still be based on a comprehensive evaluation of security patches, licenses, corporate compliance, upgrade strategies, and official support.

Image Source Process:

    1. First, pull the minio/minio and minio/mc images from the official fixed versions.
    2. Re-tag them to registry.cn-hangzhou.aliyuncs.com/pub-syq.
    3. Push them to the user's Alibaba Cloud image repository.
    4. Subsequent notes will default to using the user's Alibaba Cloud image address to ensure stability when pulling from the domestic network environment.

---

## V. Comparison Summary of MinIO and Ceph RGW

### 5.1 Similarities

Both MinIO and Ceph RGW are designed for object storage purposes.

Similarities:

    Both support the S3 API.
    Both use the Bucket / Object model.
    Both utilize AccessKey / SecretKey for authentication.
    Both are suitable for storing images, attachments, backups, and archives.
    Both offer HTTPS unified access through Nginx / Load Balancers.
    Both require mechanisms for permission management, monitoring alerts, capacity control, and backup and migration.

---

### 5.2 Differences

| Comparison Item | MinIO | Ceph RGW |
|---|---|---|
| System Focus | Dedicated to Object Storage | An object interface within the Ceph unified storage framework |
| Underlying Mechanism | MinIO's own Erasure Coding | Ceph's RADOS |
| Deployment Complexity | Relatively Lower | Depends on a complete Ceph cluster |
| Operational Complexity | Lower than Ceph | Higher |
| Storage Types | Primarily Object Storage | Also supports RBD, CephFS, and RGW |
| Common Access Methods | S3 API / Console | S3 / Swift API |
| Suitable Use Cases | Private S3 alternatives, lightweight object storage, application attachments, and backups | For scenarios where a Ceph cluster is already in place and an object interface is needed |
| Key Learning Areas | S3, EC, mc, Nginx, permissions, backup | RADOS, Pool, PG, CRUSH, RGW |

---

### 5.3 Advantages of Learning MinIO After Ceph

After having learned Ceph, it becomes easier to understand MinIO:

    You already have an understanding of object storage concepts.
    You are familiar with the Bucket and Object models.
    You know how to use AccessKey / SecretKey for authentication.
    You understand how to implement HTTPS unified access.
    You recognize the importance of monitoring, alerts, backup, and recovery processes.
    You understand that "replica/erasure coding is not equivalent to backup."
    You have a grasp of distributed storage concepts such as fault domains, capacity management| Fault Phenomenon | First Entrance | Second Entrance | Further Investigation |
|---|---|---|---|
| API Unreachable | curl health ready | Nginx error.log | Backend, port, firewall |
| Console Unreachable | curl console domain name | Nginx console configuration | 9001, redirection, WebSocket |
| mc alias Failure | Check endpoint | AccessKey / SecretKey | Certificate, 9000, permissions |
| Upload Failure | mc cp small file test | Nginx error.log | Permissions, capacity, write quorum |
| Large File Upload Failure | Nginx error.log | client_max_body_size | proxy timeout, buffer |
| 502 | Nginx error.log | Backend health check | Upstream server down |
| 413 | Nginx configuration | client_max_body_size | Upload limit issue |
| 504 | Nginx timeout | Backend performance issues | Network, disk, timeout |
| AccessDenied | User information | Policy settings | Action, Resource, Prefix |
| SignatureDoesNotMatch | Host header | Endpoint | HTTP/HTTPS, timestamp, signature verification |
| Node Offline | mc admin info | docker ps/logs | Host, Docker, network status |
| Disk Offline | mc admin info | df/lsblk/dmesg | Disk mounting, hardware issues, permissions |
| Sudden Increase in Bucket Capacity | mc du | mc find | Business activities, abnormal uploads, lifecycle management |
| Backup Failure | Mirror log checks | Alias 설정 / Permission issues | Network connectivity, target capacity, permission restrictions |

---

## IX. Summary of Production Risks for MinIO

### 9.1 Data Risks

Common risks include:

- Accidental deletion of Buckets
- Accidental deletion of Prefixes
- Accidental deletion of objects
- Use of `mc rm --recursive --force`
- Use of `mc mirror --remove`
- Application errors leading to object overwriting
- Exposure of AccessKeys resulting in malicious deletions
- Misunderstanding Erasure Coding as a form of backup
- Lack of recovery drills for backups

Mitigation strategies include:

- Backing up critical Buckets.
- Approving high-risk deletion operations.
- Implementing minimal permission policies for Policies.
- Being cautious when granting `DeleteObject` permissions.
- Enabling logging for backup tasks.
- Conducting regular recovery drills.
- Evaluating the need for version control or replication mechanisms when necessary.

---

### 9.2 Security Risks

Common risks include:

- Using the root user for business purposes
- Submitting AccessKeys/SecretKeys via Git
- Exposing HTTP port 9000 to the public internet
- Exposing Console port 9001 to the public internet
- Lacking source control over Console access
- Granting excessive permissions to business users
- Using broad policies like `s3:*` or `Resource:*
- Sharing a single set of keys across multiple businesses
- Failing to rotate keys after they are compromised

Mitigation strategies include:

- Limiting the root user's role to administrative tasks only.
- Assigning separate accounts for business users.
- Implementing minimal permission policies for Policies.
- Enforcing HTTPS for external access.
- Restricting Console access based on source IP addresses.
- Storing keys securely and not in a centralized database.
- Ensuring regular key rotation.
- Disabling compromised accounts immediately.

---

### 9.3 Capacity Risks

Common risks include:

- Approaching disk capacity limits
- Abnormal growth of Buckets
- Excessive number of small objects
- Failure to clean up temporary files
- Duplication of backup files
- Lack of a lifecycle management strategy for logs
- Focusing only on `df` reports without monitoring Bucket usage
- Delay in planning capacity expansions

Mitigation strategies include:

- Establishing clear ownership rules for Buckets.
- Conducting daily capacity checks.
- Monitoring the growth trend of Buckets.
- Setting up alerts for exceeding capacity thresholds.
- Evaluating and implementing lifecycle management policies.
- Planning capacity expansions in advance.
- Managing backup data separately from regular data.

---

### 9.4 Operational and Maintenance Risks

Common risks include:

- Lack of monitoring tools
- No alarm mechanisms in place
- Failure to collect logs properly
- Using a single Nginx server without redundancy
- Expired certificates
- Neglecting to maintain entry configuration settings
- Failing to approve high-risk operations
- Lacking post-fault analysis procedures
- Ignoring failures of backup tasks

Mitigation strategies include:

- Implementing Prometheus for monitoring.
- Using Grafana for visualizing data.
- Setting up Alertmanager for timely alerts.
- Ensuring Nginx has high availability.
- Monitoring certificate expiration dates in advance.
- Setting up alerts for backup task failures.
- Requiring approval for all high-risk operations.
- Establishing standardized fault handling procedures.

---

## X. Acceptance Checklist| Target Applications | Tools Used: Applications, SDKs, Backup Tools | Targets: Pods, StatefulSets |
| Key Features | S3 Compatibility, Object Management, Backup, Permission Control | PVCs, Replicas, Volume Recovery, Kubernetes CSI |

After learning MinIO, approaching Longhorn requires a different mindset:

    MinIO focuses on objects and S3 integration.
    Longhorn emphasizes volumes, replicas, nodes, and Kubernetes CSI support.
    MinIO is suitable for storing attachments, images, and archives.
    Longhorn is better suited for stateful application volumes.

---

## Chapter 12: Interview Response Summary

If asked during an interview:

    To what extent are you familiar with MinIO?

You can respond as follows:

    I have systematically studied and organized MinIO from its basics to production operations. My understanding goes beyond simply knowing how to start a single-instance service using docker run. I understand that MinIO is an object storage system compatible with the S3 API, with core concepts including Bucket, Object, Prefix, AccessKey, and SecretKey. It differs from block storage and file storage; typically, applications use the S3 API, mc, aws cli, or SDKs to upload and download objects.

At the deployment level, I can distinguish between various setup patterns: single-machine single-disc, single-node multi-disc, and multi-node multi-disc. A single machine is suitable for learning purposes, while a single-node multi-disc configuration allows for understanding Erasure Coding concepts. However, for production use, a multi-node multi-disc setup is highly recommended. I have planned a distributed MinIO cluster with 4 nodes and multiple disks, ensuring that the startup endpoint list for each node is consistent and that data directories are stored on separate disks to prevent accidental writes to system disks.

In terms of access design, I separate internal communication from external access. MinIO nodes can communicate within a trusted intranet using HTTP, just like Nginx communicating with backend nodes. However, external clients must use HTTPS for secure access. The API and Console entries should also be separated; for example, s3.example.com could proxy port 9000, while minio-console.example.com proxies port 9001. Additionally, the Console should only be accessible from the operations network segment.

Regarding permission management, I do not assign the root user to business purposes. Instead, I create separate users and Policies for each business or environment. Policies implement fine-grained access control based on Buckets or Prefixes, with read, write, and delete permissions designed separately. AccessKeys and SecretKeys are managed at the key level; they should not be stored in Git, and in case of leakage, the user account should be immediately disabled, logs should be reviewed, and the keys rotated.

For data protection, I understand that MinIO uses Erasure Coding to handle disk or node failures. I am also familiar with the concepts of read quorum and write quorum. In the event of a node or disk issue, I would start my investigation by checking the mc admin info, health interfaces, Docker logs, disk mount status, and Nginx logs. Additionally, I would use the `mc admin heal --dry-run` command to assess the possible recovery scope. However, it’s important to note that Erasure Coding is not a backup solution and cannot prevent accidental deletions or entire cluster failures.

For production operations, I would integrate Prometheus and Grafana to monitor key indicators such as node health, disk usage, Bucket capacity, object count, S3 request rates, error codes (4xx/5xx), API response times, Nginx traffic, certificate expiration dates, and backup tasks. For backup and migration purposes, I would use `mc mirror` to schedule regular backups or perform cross-cluster migrations. Before executing such operations, I would always conduct a dry-run test and carefully handle parameters like `--overwrite` and `--remove`. After the migration, I would verify the capacity, object count, and the integrity of critical objects.

In summary, my understanding of MinIO goes beyond its installation process. From a senior SRE perspective, I consider various aspects including deployment, security, permission control, monitoring, fault recovery, backup, and production readiness.

---

## Chapter 13: Summary of This Article

This article provides a comprehensive overview of the MinIO framework:

1. MinIO is an object storage system compatible with the S3 API.
2. It is well-suited for storing unstructured data such as images, attachments, backups, archives, logs, and package materials.
3. The core concepts of MinIO include Bucket, Object, and Prefix.
4. The default ports for S3 API interactions are 9000, and the Web Console uses port 9001.
5. A single-machine single-disc setup is suitable for learning but not for production use.
6. Single-node multi-disc configurations can utilize Erasure Coding, though they do not provide robust fault tolerance.
7. Multi-node multi-disc setups are more representative