# RustFS Directory Index

Recommended Path: 05-Storage/04-RustFS/00-RustFS Directory Index.md

Tags: #RustFS #Object Storage #S3 #MinIO Alternative #Distributed Storage #Docker #Reverse Proxy #HTTPS #mc #AWSCLI #Advanced SRE #Production Operations

---

## I. Module Overview

RustFS is an object storage system compatible with S3, written in Rust.

Its learning objectives are:

    To serve as a supplementary module to MinIO for object storage.
    To focus on understanding the deployment, usage, operations, and production boundaries of new S3-compatible object storage solutions.
    To conduct experiments using both single-machine Docker setups and multi-node Docker/VM clusters.
    To pay special attention to S3 APIs, Buckets, Objects, AccessKeys, SecretKeys, HTTPS access, reverse proxies, logging, capacity management, fault recovery, and ecological compatibility.

Like MinIO, RustFS belongs to the category of object storage solutions. However, it is not:

    Block storage
    File system storage
    Kubernetes CSI block storage
    A substitute for Longhorn
    A substitute for Ceph RBD
    A general-purpose NFS substitute

Its main purposes include:

    Enabling applications to upload and download objects through S3 APIs
    Providing private object storage services
    Storing unstructured data such as images, attachments, backup packages, log archives, AI datasets, and model files
    Serving as a substitute, compatible alternative, migration tool, or comparative study for MinIO-like object storage systems

---

## II. Learning Goals for This Module

After completing this RustFS module, you should be able to:

1. Understand the basic positioning of RustFS.
2. Comprehend the relationship between RustFS and the S3 object storage model.
3. Recognize the similarities and differences between RustFS and MinIO.
4. Deploy a single-machine RustFS service using Docker.
5. Plan the data directories, access ports, and entry points for RustFS.
6. Access RustFS using S3 clients.
7. Use tools like mc or AWS CLI to create Buckets, upload objects, download objects, and verify access.
8. Plan the nodes, disks, and access entry points for a RustFS cluster deployment.
9. Design internal HTTP communication mechanisms and external HTTPS access methods.
10. Set up Nginx as a reverse proxy for RustFS API access.
11. Understand AccessKeys/SecretKeys, Bucket permissions, and access control.
12. Configure or plan HTTPS settings, certificates, and unified entry points.
13. Monitor logs, check capacity levels, and monitor node status.
14. Troubleshoot issues such as failed service startups, port conflicts, permission errors, and S3 access failures.
15. Understand the production suitability boundaries of RustFS as a new object storage solution.
16. Determine whether RustFS is suitable for testing, alternative verification, or production deployment.
17. Compare RustFS with MinIO, Ceph RGW, and cloud vendors' OSS/S3 solutions.
18. Establish advanced SRE operation and maintenance methodologies for the RustFS module.

---

## III. Experimental Implementation Methods

The experimental methods for this RustFS module include:

    Single-machine Docker deployment
    Multi-node Docker/VM cluster deployment

These methods do not rely on Kubernetes, Longhorn, or any modifications to containerd. The same experimental IP range (10.0.0.0/24) is used as before.

Addresses already occupied or reserved:

| IP | Purpose |
|---|---|
| 10.0.0.10 | Operations server, GitLab/Jenkins/Harbor/ops services |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |
| 10.0.0.41 - 10.0.0.46 | MinIO experimental nodes and entry points |

It is recommended to plan separate addresses for RustFS experiments to avoid mixing them with Ceph, MinIO, or Longhorn experiments.

RustFS experimental node planning:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Node 1 |
| 10.0.0.52 | rustfs-node02 | RustFS Node 2 |
| 10.0.0.53 | rustfs-node03 | RustFS Node 3 |
| 10.0.0.54 | rustfs-node04 | RustFS Node 4 |
| 10.0.0.55 | rustfs-client | S3 client/mc/aws cli testing node |
| 10.### 08-RustFS Operations and Troubleshooting: Logs, Capacity, Node Exceptions, and Recovery

File:

    08-RustFS Operations and Troubleshooting: Logs, Capacity, Node Exceptions, and Recovery.md

Key Points:

    Service log monitoring.
    Docker container status checks.
    Data directory capacity management.
    Handling node disk and port exceptions.
    Troubleshooting S3 access failures, AccessDenied errors, and SignatureDoesNotMatch issues.
    Restoration procedures for node failures and data directory deletions.
    Establishment of routine inspection commands and production failure record templates.

Practical Goals:

    Be able to diagnose RustFS container startup failures.
    Identify and resolve port occupation issues.
    Adjust data directory permissions correctly.
    Resolve client access problems.
    Debug Nginx reverse proxy errors.
    Address abnormal Bucket/Object access cases.
    Develop efficient daily inspection routines.
    Maintain detailed production fault records.### Basic Operations on Buckets and Objects
Data persistence verification
Client access verification

Not used for:

High availability in production environments
Node failure recovery
Multi-node disaster recovery verification

---

### 8.2 Multi-Node Mode Topology

Multi-node Docker/VM cluster experiment:

    ┌───────────────────────────────┐
    │ rustfs-node01 10.0.0.51       │
    │ /data/rustfs                  │
    └───────────────┬───────────────┘
                    │
    ┌───────────────┼───────────────┐
    │               │               │
    v               v               v
    ┌───────────────────────────────┐
    │ rustfs-node02 10.0.0.52       │
    │ /data/rustfs                  │
    └───────────────────────────────┘

    ┌───────────────────────────────┐
    │ rustfs-node03 10.0.0.53       │
    │ /data/rustfs                  │
    └───────────────────────────────┘

    ┌───────────────────────────────┐
    │ rustfs-node04 10.0.0.54       │
    │ /data/rustfs                  │
    └───────────────────────────────┘

    ┌───────────────────────────────┐
    │ rustfs-entry 10.0.0.56        │
    │ Nginx HTTPS unified entry          │
    └───────────────────────────────┘

    ┌───────────────────────────────┐
    │ rustfs-client 10.0.0.55       │
    │ mc / aws cli / app            │
    └───────────────────────────────┘

---

### 8.3 Access Gateway Design

Recommended access strategies:

Internal node communication: HTTP
External client access: HTTPS
Unified gateway: Nginx/Load Balancer
Clients should only access via a unified domain name
Backend node ports should not be directly exposed to the public network

Example:

rustfs-api.example.com -> Nginx HTTPS -> RustFS API backend nodes

Experimental domain names can be:

rustfs.local
s3.rustfs.local

Local hosts example:

10.0.0.56 s3.rustfs.local

---

## IX. Port Planning

Specific port numbers should be determined according to subsequent RustFS official documentation and actual startup parameters.

Principles for this module's port planning:

| Port Type | Description | Exposure Recommendation |
|---|---|---|
| RustFS API Ports | S3 API access ports | Exposed on the internal network, with external access via Nginx |
| RustFS Console/Management Ports | If the current version provides a management interface | Source restricted, not directly exposed to the public network |
| Nginx 443 | External HTTPS unified gateway | Open to clients |
| Nginx 80 | Can be used for redirecting to HTTPS | Optional |
| SSH 22 | For node operations and maintenance | Only accessible from designated maintenance networks |

Production principles:

API ports should not be directly exposed to the public network.
Management ports should not be directly exposed to the public network.
All external access must use HTTPS.
Internal HTTP access is limited to trusted networks only.
Firewalls should restrict the sources of access requests.
Port planning details should be documented to avoid conflicts.

---

## X. Data Directory Planning

For single-node mode:

/data/rustfs

For multi-node mode:

/data/rustfs

If multiple disks are used, the directories can be organized as follows:

/data/rustfs/disk1
/data/rustfs/disk2
/data/rustfs/disk3
/data/rustfs/disk4

Recommended data disk practices:

Use dedicated data disks.
Never use system disks to store production data.
Choose file systems like xfs or ext4.
Mount the disks and add them to /etc/fstab.
Use the noatime mount option if appropriate.
Ensure that the capacity of each disk is consistent across all nodes.
Try to maintain similar disk performance levels.

Check commands:

df -hT
lsblk -f
mount | grep /data
du -sh /data/rustfs

Important warnings:

Never manually delete the RustFS data directory.
Do not use the command rm -rf /data/rustfs.
Do not mix the data directory with other business logs or caches.
Abnormal permissions in the data directory can cause service failures or object write errors.

---

## XI. Image Management and Domestic Network Optimization

### 11.1 Principles for Image Management

The RustFS module will be deployed using Docker in the future.

Image management principles include### Conduct fault simulations again.
### Perform performance stress testing once more.
### Finally, evaluate whether it is suitable for production pilots.

---

## Section Fifteen: The Relationship Between RustFS and Advanced SRE Capabilities

The value of learning RustFS lies not just in being able to launch an object storage container. More importantly, it helps to develop the following capabilities:

- Ability to assess new infrastructure components
- Capability to verify S3 compatibility
- Ability to determine the production boundaries of object storage
- Ability to deploy Docker on both single machines and multiple nodes
- Ability to design reverse proxies and HTTPS gateways
- Ability to manage AccessKey/SecretKey permissions effectively
- Ability to maintain Bucket/Object operations
- Proficiency with the mc/AWS CLI toolchain
- Skills in handling logs, monitoring capacity, managing ports, and troubleshooting failures
- Capability to evaluate the migration of old and new object storage systems
- Ability to compare and select between RustFS and solutions like MinIO/Ceph RGW

From the perspective of an advanced SRE, the focus of the RustFS module is not simply “replacing MinIO” but rather:

- How to validate new technologies,
- How to conduct stress tests,
- How to assess risks,
- And how to design appropriate production boundaries.

---

## Section Sixteen: Requirements for Subsequent RustFS Notes

Each subsequent RustFS note must meet the following criteria:

- Be modular and independent, allowing for standalone reading.
- Each note should have a clear experimental objective.
- Include actual command examples.
- Be feasible to implement in Docker/VM environments.
- Not rely on Kubernetes.
- Avoid interference with experiments involving MinIO, Ceph, or Longhorn.
- Use the IP range 10.0.0.0/24 for independent planning.
- Utilize fixed version images.
- Record the source of the images used.
- Demonstrate both internal HTTP and external HTTPS configurations.
- Show how Nginx is used as a unified entry point.
- Clearly outline port management strategies.
- Explain data directory organization.
- Address permission and security considerations.
- Include details on log management, capacity monitoring, and fault troubleshooting.
- Highlight production boundary considerations.
- Refer to official documentation wherever appropriate.

---

## Section Seventeen: Phase-by-Phase Implementation Suggestions

It is recommended to proceed in the following order:

- Step 1: Clearly explain the basics of RustFS, S3 object storage, and its applicable scenarios.
- Step 2: Distinguish between single-machine mode and cluster mode.
- Step 3: Complete the deployment and access verification of RustFS on a single Docker machine.
- Step 4: Implement multi-node, multi-disk setups with a unified entry point.
- Step 5: Compare RustFS with MinIO in terms of system functionality.
- Step 6: Use mc/AWS CLI to establish client connectivity.
- Step 7: Complete the design of HTTPS, reverse proxies, and permission security mechanisms.
- Step 8: Ensure proper log management, capacity monitoring, and fault recovery procedures for node abnormalities.
- Step 9: Conduct a comprehensive phase summary and determine the appropriate production boundaries.

---

## Section Eighteen: Interview Response Guidelines

If you are asked in an interview:

- What is RustFS? What’s your opinion about it?

You can respond as follows:

- RustFS is an S3-compatible object storage system developed using Rust. It is similar to MinIO in its purpose, targeting use cases such as storing images, attachments, backup files, log archives, AI datasets, and model files for unstructured data. I would start learning about RustFS after studying MinIO first, because MinIO’s object storage framework, mc tools, Bucket management, AccessKey security, reverse proxy mechanisms, HTTPS support, and backup strategies can all serve as a valuable foundation for understanding RustFS.
- However, I wouldn’t consider RustFS a direct replacement for mature production solutions like MinIO or Ceph RGW. As a new object storage option, it requires thorough testing in terms of version stability, S3 API compatibility, Multipart Upload support, SDK integration, permission management, log monitoring, node failure recovery, backup capabilities, and practical use cases.
- In my experiments, I would first deploy RustFS on a single Docker machine to verify service startup, data directory setup, port configuration, Bucket creation, and file upload/download operations. Then, I would move on to deploying it in a multi-node Docker/VM cluster to test node performance, disk management, unified entry point functionality, and fault recovery mechanisms. Finally, I would use Nginx to provide an external HTTPS interface, while internal communication between nodes could utilize HTTP over a secure network.
- From an operational perspective, my focus would be on managing logs, monitoring port activity, ensuring adequate capacity, handling AccessDenied errors, SignatureDoesNotMatch issues, and resolving Nginx 502/413/504 errors. I would also pay close attention to certificate and key