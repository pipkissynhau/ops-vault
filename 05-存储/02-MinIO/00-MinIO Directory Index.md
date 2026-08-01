# MinIO Directory Index

Recommended path: 05-Storage/02-MinIO/00-MinIO Directory Index.md

Tags: #MinIO #ObjectStorage #S3 #ErasureCoding #DistributedStorage #Docker #Nginx #HTTPS #AdvancedSre #ProductionTransport

---

## I. Module Overview

This document serves as a directory index for the MinIO module, outlining a complete learning path from foundational theory, deployment practices, distributed clusters, unified entry points, permission management, data protection, monitoring operations, to stage summaries.

MinIO is an object storage system compatible with the S3 API, suitable for:

    Image storage
    Attachment storage
    Backup archiving
    Log archiving
    Private object storage
    DevOps toolchain artifact storage
    Object data storage in AI/data platforms

This module is positioned as:

    A MinIO learning module independent of Ceph, Longhorn, RustFS.
    Each note can be read and experimented with independently.
    Experiments primarily use Docker single-node, Docker multi-node, and VM multi-node clusters.
    Focuses on advanced SRE understanding of object storage architecture, deployment implementation, fault recovery, permission governance, entry design, monitoring alerts, and production operations capabilities.

---

## II. Module Learning Objectives

After completing the MinIO module, you should be able to:

1. Understand the basic concepts of object storage, S3 protocol, Bucket, Object, AccessKey, and SecretKey.
2. Understand the differences between MinIO and traditional file systems, block storage, Ceph RGW, and cloud vendor OSS/S3.
3. Understand MinIO's Erasure Coding data protection mechanism.
4. Master the method of deploying MinIO using single-node Docker.
5. Master the method of experimental deployment for single-node multi-disk MinIO.
6. Master the method of deploying a distributed MinIO cluster with multiple nodes and disks.
7. Understand MinIO's production design using HTTP for internal node communication and HTTPS for external client access.
8. Be able to provide a unified HTTPS entry for MinIO using Nginx.
9. Be able to correctly plan the 9000 API port, 9001 Console port, and reverse proxy entry.
10. Master mc client configuration, Bucket management, object upload/download, and policy management.
11. Master MinIO user, policy, AccessKey, and SecretKey permission governance methods.
12. Understand MinIO node failure, disk failure, and Erasure Coding recovery logic.
13. Master daily monitoring methods for Prometheus metrics, logs, capacity, Bucket/Object growth, etc.
14. Master mc mirror, cross-cluster synchronization, and data migration concepts.
15. Be able to determine if a MinIO cluster meets basic production readiness criteria.

---

## III. Experiment Environment Planning

### 3.1 Experiment Network Segment

This module's experiment environment uses:

    10.0.0.0/24

Note to avoid existing addresses:

| IP | Used Objects |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |

---

### 3.2 MinIO Experiment Node Planning

Recommended node planning:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO single-node / distributed node 1 |
| 10.0.0.42 | minio-node02 | MinIO distributed node 2 |
| 10.0.0.43 | minio-node03 | MinIO distributed node 3 |
| 10.0.0.44 | minio-node04 | MinIO distributed node 4 |
| 10.0.0.45 | minio-client | mc client / stress testing / backup synchronization |
| 10.0.0.46 | minio-entry | Nginx unified entry / HTTPS reverse proxy (optional) |

Notes:

    Single-node experiments can use only 10.0.0.41.
    Single-node multi-disk experiments can use only 10.0.0.41.
    4-node multi-disk distributed experiments use 10.0.0.41-10.0.0.44.
    Unified entry experiments use 10.0.0.46 or directly deploy Nginx on minio-node01.

---

### 3.3 Operating System Recommendations

Recommended system:

    Ubuntu Server 22.04.5 LTS

Optional systems:

    Rocky Linux 9

This module's commands default to Ubuntu Server 22.04.5 LTS.

Rocky Linux 9 will have systemd, firewalld, and package installation differences supplemented when needed.

---

### 3.4 Container Runtime Method

MinIO module experiments primarily use:

    Docker

Do not directly modify Kubernetes containerd runtime.

Reasons:

    This module primarily trains object storage itself.
    Docker deployment is more suitable for quickly understanding single-node, multi-disk, and multi-node cluster structures.
    Object storage in Kubernetes is not necessarily accessed via CSI, applications typically access via S3 API.
    Not disrupting the underlying runtime aligns with advanced SRE experimental and production methodologies.

---

## IV. Image Version Planning

### 4.1 MinIO Server Image

This module defaults to using a fixed version image already synchronized to the Alibaba Cloud registry:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Source image:

    minio/minio:RELEASE.2025-04-22T22-12-26Z

---

### 4.2 mc Client Image

This module defaults to using a fixed version image already synchronized to the Alibaba Cloud registry:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Source image:

    minio/mc:RELEASE.2025-04-16T18-13-26Z

---

### 4.3 Reason for Using Fixed Versions

This module does not use latest, but instead uses fixed versions, for the following reasons:

1. Avoid changes to 'latest' causing inconsistencies in the Web Console, command parameters, log formats, or functional behavior.
2. Ensure reproducibility of experiments including single-node Docker, single-node multi-disk, multi-node distributed, reverse proxy, permissions, Bucket, Policy, mc management, backup migration, etc.
3. MinIO server uses RELEASE.2025-04-22T22-12-26Z to maintain consistency in Web Console management capabilities, S3 API behavior, and experimental documentation during the learning phase.
4. mc uses RELEASE.2025-04-16T18-13-26Z to keep the client-server version close in time, reducing capability mismatches between client and server.
5. Production environment version selection still requires comprehensive evaluation based on security patches, licenses, enterprise compliance, upgrade strategies, and official support.

---

### 4.4 Image Source Record

The image source process for this module is as follows:

    1. Pull MinIO server image from official fixed version:
       minio/minio:RELEASE.2025-04-22T22-12-26Z

    2. Pull mc client image from official fixed version:
       minio/mc:RELEASE.2025-04-16T18-13-26Z

    3. Retag to user's Alibaba Cloud image repository:
       registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
       registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

    4. Push to Alibaba Cloud image repository for stable pull in domestic network environments.

---

### 4.5 Current Available Images

Current ops-server has the following images:

    minio/mc:RELEASE.2025-04-16T18-13-26Z
    minio/minio:RELEASE.2025-04-22T22-12-26Z
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Subsequent MinIO notes will default use:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

---

## FiveI don't know.Network and Port Planning

### 5.1 MinIO Ports

Common MinIO ports:

| Port | Function |
|---|---|
| 9000 | S3 API server port |
| 9001 | Web Console management port |
| 443 | Unified HTTPS external entry |
| 80 | Optional, for HTTP redirect to HTTPS |

---

### 5.2 Module Port Design

Internal node communication:

    MinIO nodes use HTTP between each other.
    Internal communication primarily uses port 9000.
    Internal network resides in the trusted 10.0.0.0/24 experimental subnet.

Client public or external access:

    Must use HTTPS.
    Recommended to expose unified entry through Nginx / LB.
    External entry uses port 443.
    Not recommended to expose port 9000 HTTP directly to public network.

---

### 5.3 Internal HTTP and External HTTPS Strategy

This module adopts the following strategy:

    Internal node communication uses HTTP.
    External client access uses HTTPS.
    Reverse proxy provides unified entry.
    Nginx proxies MinIO 9000 API port.
    Console 9001 can be accessed via independent domain or controlled operations entry as needed.

Reasons:

    Internal nodes reside in trusted network, using HTTP reduces TLS complexity and encryption/decryption overhead.
    External access involves user credentials, object data, and S3 API requests, which must use HTTPS.
    Erasure Coding, data replication, and high availability capabilities do not depend on internal HTTPS.
    If enterprise compliance requires encryption for internal links, further evaluation of full-chain TLS is recommended.

---

### 5.4 Unified Entry Design

Recommended structure:

    Client
      |
      | HTTPS 443
      v
    Nginx / LB
      |
      | HTTP 9000
      v
    MinIO distributed cluster

Diagram:

    ┌──────────────┐
    │   Client     │
    └──────┬───────┘
           │ HTTPS 443
           v
    ┌──────────────┐
    │ Nginx / LB   │
    │ minio-entry  │
    └──────┬───────┘
           │ HTTP 9000
           v
    ┌──────────────────────────────────┐
    │          MinIO Cluster            │
    │ minio-node01 / 02 / 03 / 04       │
    └──────────────────────────────────┘

---

## SixI don't know.Directory Planning

MinIO module directories are as follows: /think

## VII. Notes Structure Planning

### 7.1 00-MinIO Directory Index

Target:

    Establish a MinIO module learning path.
    Plan experimental nodes, ports, images, deployment methods, and productionization goals.

Focus:

- Module positioning
- Experimental environment
- Image version
- Directory planning
- Learning route
- Productionization goals

---

### 7.2 01-MinIO Basics: Object Storage, S3 Protocol, and Erasure Coding

Target:

    Understand the basic model of MinIO.

Focus:

- What is object storage
- Bucket / Object / Prefix
- Basic concepts of S3 protocol
- AccessKey / SecretKey
- Erasure Coding
- Difference between MinIO and file storage, block storage
- Difference between MinIO and Ceph RGW

---

### 7.3 02-MinIO Deployment Modes: Single Machine, Single Node Multi-Disk, and Multi-Node Cluster

Target:

    Understand the applicable boundaries of different MinIO deployment modes.

Focus:

- Single machine single disk
- Single machine multi-disk
- Single node multi-disk Erasure Coding
- Multi-node multi-disk distributed cluster
- Differences between development testing and production deployment
- Capacity, node, and disk planning

---

### 7.4 03-MinIO Distributed Cluster Deployment: 4-Node Multi-Disk Experiment Practice

Target:

    Complete a 4-node distributed MinIO cluster experiment.

Focus:

- 4-node planning
- Data directory per node
- Docker startup command
- Distributed server address
- Console access
- mc alias configuration
- Bucket creation
- Upload/download verification
- Initial observation of node failure

---

### 7.5 04-MinIO Access Entry Design: Internal HTTP Communication and External HTTPS Access

Target:

    Design a MinIO access architecture closer to production.

Focus:

- Internal node HTTP communication
- External HTTPS access
- Why not recommend exposing 9000 HTTP directly
- Separation of API entry and Console entry
- Domain planning
- Certificate planning
- Security boundaries

---

### 7.6 05-MinIO Reverse Proxy: Nginx Unified Entry, Certificates, and 9000 Port Proxy

Target:

    Use Nginx to provide MinIO HTTPS unified entry.

Focus:

- Nginx upstream
- HTTPS certificate
- 9000 API proxy
- 9001 Console proxy
- client_max_body_size
- proxy_buffering
- Host header
- S3 signature and reverse proxy considerations

---

### 7.7 06-MinIO Client Tools: mc Configuration, Bucket Management, and Object Operations

Target:

    Master the common management capabilities of the mc tool.

Focus:

- mc alias set
- mc admin info
- mc mb
- mc ls
- mc cp
- mc mirror
- mc rm
- mc stat
- mc du
- Bucket management
- Object upload/download
- Daily troubleshooting

---

### 7.8 07-MinIO Access Control: Users, Policy, AccessKey, and SecretKey

Target:

    Master MinIO access control governance methods.

Focus:

- Root user boundaries
- Regular users
- AccessKey / SecretKey
- Policy
- Bucket-level permissions
- Read-only / read-write permissions
- Key leakage handling
- Principle of least privilege
- Production security considerations

---

### 7.9 08-MinIO Data Protection: Erasure Coding, Node Failure, and Disk Failure Recovery

Target:

    Understand MinIO data protection and failure recovery.

Focus:

- Erasure Coding working mechanism
- Node failure
- Disk failure
- Availability boundaries
- Data repair
- healing
- mc admin heal
- Failure drill
- Production considerations

---

### 7.10 09-MinIO Monitoring and Operations: Prometheus Metrics, Logs, and Capacity Management

Target:

    Establish MinIO daily operations capabilities.

Focus:

- Health check
- Log viewing
- Prometheus metrics
- Grafana dashboard
- Bucket capacity
- Object count
- Disk capacity
- Node status
- Alarm suggestions
- Daily inspection

---

### 7.11 10-MinIO Backup and Migration: mc mirror, Cross-Cluster Synchronization, and Data Migration

Target:

    Master MinIO data backup and migration methods.

Focus:

- mc mirror
- One-way synchronization
- Cross-cluster synchronization
- Data migration
- Bucket migration
- From MinIO to MinIO
- From S3 to MinIO
- From MinIO to S3
- Migration verification
- Production considerations

---

### 7.12 99-MinIO Stage Summary

Target:

    Summarize the complete capabilities of MinIO from basics to production-ready cluster.

Focus:

- Object storage model
- S3 access methods
- Single machine and distributed clusters
- Erasure Coding
- Unified entry
- Access control governance
- Monitoring and operations
- Backup and migration
- Comparison with Ceph RGW / RustFS
- Production acceptance

---

## VIII. MinIO vs Ceph RGW Comparison

### 8.1 Commonalities

MinIO and Ceph RGW both provide object storage capabilities.

Commonalities:

- Both are compatible with S3 API.
- Both have Bucket / Object.
- Both use AccessKey / SecretKey.
- Both can be used for image, attachment, backup, and archive scenarios.
- Both can provide HTTPS unified entry through Nginx / LB.
- Both require attention to permissions, capacity, monitoring, backup, and security.

---

### 8.2 Differences /think

| Comparison Item | MinIO | Ceph RGW |
|---|---|---|
| Position | Focused on object storage | Object interface within Ceph's unified storage system |
| Underlying | MinIO's own Erasure Coding | RADOS |
| Interface | S3 as core | S3 / Swift |
| Deployment | Relatively simple | Depends on a complete Ceph cluster |
| Operation Complexity | Lower than Ceph | High |
| Storage Type | Object storage | Ceph also has RBD / CephFS / RGW |
| Use Cases | Private S3, lightweight object storage | Existing Ceph cluster needing object interface |
| Learning Difficulty | Relatively low | Relatively high |

---

### 8.3 Significance of Learning Order

After learning Ceph, learning MinIO will be easier because:

    Already understand object storage.
    Already understand Bucket / Object.
    Already understand AccessKey / SecretKey.
    Already understand HTTPS unified entry point.
    Already understand the importance of capacity, monitoring, backup, and fault recovery.
    Already understand that productionization isn't just deploying successfully.

Key areas to focus on with MinIO modules:

    MinIO's own deployment mode.
    Erasure Coding.
    mc tool.
    Permission policies.
    Nginx entry point.
    Data synchronization and migration.
    Distributed cluster fault recovery.

---

## NineI don't know.MinIO Production Focus Points

### 9.1 Architecture Planning

Must clarify before production:

- Number of nodes
- Number of disks
- Single node multi-disk or multi-node multi-disk
- Network bandwidth
- Internal communication method
- External access entry point
- Whether API and Console are separated
- Certificates and domain names
- Backup and synchronization strategies
- Monitoring and alerting strategies

---

### 9.2 High Availability

Production focus:

- Whether node failure is tolerable
- Whether disk failure is tolerable
- Whether Erasure Coding ratio is reasonable
- Whether Nginx / LB is highly available
- Whether Console needs independent access restrictions
- Whether the cluster has single-point entry risks

---

### 9.3 Data Protection

Production focus:

- Erasure Coding is not equal toAlien.backup
- Node failure recovery process
- Disk failure recovery process
- mc mirror backup synchronization
- Cross-cluster backup
- Regular recovery verification
- Delete protection and permission control

---

### 9.4 Security

Production focus:

- Must use HTTPS externally
- Do not expose internal HTTP 9000 directly
- Console should not be arbitrarily exposed to the public
- Root user should not be used by business
- Each business should have independent AccessKey
- Policy with minimal permissions
- AccessKey regular rotation
- Keys not submitted to Git
- Nginx access logs should be auditable

---

### 9.5 Monitoring and Operations

Production focus:

- Node online status
- Disk online status
- Disk capacity
- Bucket capacity
- Object count
- API request error rate
- API latency
- Prometheus metrics
- Log collection
- Certificate expiration
- Backup task success rate
- Data synchronization delay

---

## TenI don't know.MinIO High-Risk Operation Warnings

The following operations require caution:

    Delete Bucket
    Recursive delete objects
    Delete user
    Delete AccessKey
    Modify Policy
    Modify distributed cluster startup parameters
    Modify data directory
    Delete Docker volume or host data directory
    Reinitialize cluster
    Modify Nginx Host / proxy configuration
    Expose HTTP 9000 directly to the public

Must confirm before production environment execution:

    Whether it's a test environment.
    Whether there is a backup.
    Whether there is approval.
    Whether there is a rollback plan.
    Whether the business has been notified.
    Whether someone has reviewed.
    Whether the operation target is confirmed correctly.

---

## ElevenI don't know.Staged Learning Recommendations

Recommended learning order for MinIO modules:

    1. First understand object storage and S3.
    2. Then understand Erasure Coding.
    3. First do single-machine Docker.
    4. Then do single-node multi-disk.
    5. Then do 4-node distributed cluster.
    6. Then add Nginx HTTPS unified entry point.
    7. Then learn mc tool.
    8. Then learn users and Policy.
    9. Then do node and disk failure drills.
    10. Finally supplement monitoring, backup migration, and stage summary.

Do not start by pursuing complex production architecture.

Correct order is:

    Run it first.
    Then understand.
    Then enhance.
    Then govern.
    Then productionize.

---

## TwelveI don't know.MinIO Module Acceptance Criteria

After completing this module, the following standards should be met:

### 12.1 Basic Understanding

| Capability Item | Mastered |
|---|---|
| Can explain object storage, Bucket, Object |  |
| Can explain basic S3 API access methods |  |
| Can explain AccessKey / SecretKey |  |
| Can explain the role of Erasure Coding |  |
| Can explain the difference between MinIO and Ceph RGW |  |

---

### 12.2 Deployment Ability

| Capability Item | Mastered |
|---|---|
| Can start single-machine MinIO with Docker |  |
| Can deploy single-node multi-disk MinIO |  |
| Can deploy 4-node distributed MinIO |  |
| Can use fixed version images |  |
| Can use Alibaba Cloud image repository to improve domestic pull stability |  |
| Can plan 9000 / 9001 / 443 ports |  |

---

### 12.3 Operation Ability

| Capability Item | Mastered |
|---|---|
| Can use mc alias to manage clusters |  |
| Can create and delete Buckets |  |
| Can upload, download, and delete objects |  |
| Can view capacity and object count |  |
| Can view service logs |  |
| Can judge node or disk anomalies |  |
| Can perform basic fault recovery verification |  |

---

### 12.4 Production Governance Ability

| Capability Item                    | Mastered |
| ---------------------- | ---- |
| Can configure Nginx HTTPS unified entry point   |      |
| Can design internal HTTP, external HTTPS architecture |      |
| Can create business users and minimal permission Policy    |      |
| Can plan AccessKey rotation       |      |
| Can design Prometheus monitoring metrics    |      |
| Can design capacity and node alerts             |      |
| Can use mc mirror for backup and migration    |      |
| Can judge whether MinIO cluster meets basic production conditions |      |

---

## ThirteenI don't know.Reference Documents /think

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Docker Deployment Documentation:

    https://min.io/docs/minio/container/index.html

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO Erasure Coding Documentation:

    https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html

MinIO Distributed Deployment Documentation:

    https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html

MinIO User and Permissions Documentation:

    https://min.io/docs/minio/linux/administration/identity-access-management.html

MinIO Prometheus Monitoring Documentation:

    https://min.io/docs/minio/linux/operations/monitoring/collect-minio-metrics-using-prometheus.html

MinIO Bucket Replication / Mirror Related Documentation:

    https://min.io/docs/minio/linux/administration/bucket-replication.html

AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

Nginx Official Documentation:

    https://nginx.org/en/docs/