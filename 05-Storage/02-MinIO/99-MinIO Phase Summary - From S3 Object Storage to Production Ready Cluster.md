# MinIO Stage Summary: From S3 Object Storage to Production-Ready Cluster

Suggested Path: 05-Storage/02-MinIO/99-MinIO Stage Summary: From S3 Object Storage to Production-Ready Cluster.md

Tags: #MinIO #SummaryOfPhases #ObjectStorage #S3 #ErasureCoding #Docker #Nginx #HTTPS #Prometheus #BackupMigration #AdvancedSre #ProductionTransport

---

## I. Document Description

This document serves as a stage summary for the MinIO module, used to conclude the entire MinIO learning phase.

Previously, a complete learning path from foundational theory to production operations has been completed, including:

- MinIO Object Storage Basics
- S3 API, Bucket, Object, Prefix
- Single-Machine Docker Deployment
- Single-Node Multi-Disk Deployment
- 4-Node Multi-Disk Distributed Cluster Deployment
- Internal HTTP Communication and External HTTPS Access Design
- Nginx Unified Entry and 9000/9001 Port Proxy
- mc Client Tool
- Bucket Management and Object Operations
- User, Policy, AccessKey, SecretKey Access Governance
- Erasure Coding Data Protection
- Node and Disk Failure Recovery
- Prometheus Monitoring, Logs, and Capacity Management
- mc mirror Backup Migration and Cross-Cluster Synchronization

This document focuses on summarizing:

    What was learned in the MinIO stage
    What is MinIO's core architecture and operations logic
    How to elevate from "able to deploy" to "able to operate in production"
    Differences between MinIO and Ceph RGW
    Relationship between MinIO and future RustFS, Longhorn learning
    What conditions a production-ready MinIO cluster should have

---

## II. Stage Positioning

MinIO is an object storage system compatible with S3 API.

Its core positioning is:

    Provide private object storage capabilities
    Compete with cloud vendors' OSS/S3/COS
    Suitable for unstructured data like images, attachments, backups, archives, logs, artifacts, and AI datasets
    Access via HTTP/HTTPS API
    Organize data using Bucket/Object model

From an advanced SRE perspective, the value of learning MinIO goes beyond deploying an object storage service—it's about establishing a production operations methodology for object storage:

    How to plan nodes and disks
    How to protect object data
    How to design access entry points
    How to govern permissions and keys
    How to monitor capacity and error rates
    How to perform backups and migrations
    How to handle failures
    How to determine if a cluster meets production readiness

---

## III. Summary of MinIO Learning Mainline

### 3.1 First Mainline: Object Storage Model

Core Questions:

    What is object storage?
    What is a Bucket?
    What is an Object?
    What is a Prefix?
    What is S3 API?

Stage Gains:

    Understand MinIO is not block storage or traditional file system.
    Understand Bucket is the top-level container for object storage.
    Understand Object is the actual data.
    Understand Prefix is the prefix of Object Key, not a real directory.
    Understand S3 API is the main way applications access MinIO.
    Understand AccessKey/SecretKey are credentials for object storage access.

---

### 3.2 Second Mainline: Deployment Patterns

Core Questions:

    What are MinIO's deployment patterns?
    What is suitable for single-machine single-disk?
    What is suitable for single-node multi-disk?
    Why is multi-node multi-disk closer to production?

Stage Gains:

    Single-machine single-disk is suitable for learning, development, and functional verification.
    Single-node multi-disk helps understand Erasure Coding but cannot provide node-level high availability.
    Multi-node multi-disk is closer to production deployment.
    4-node multi-disk cluster can validate node failure, disk failure, and basic high availability capabilities.
    All nodes in MinIO distributed cluster must have consistent endpoint lists.
    Data directories must be planned in advance and cannot be mistakenly written to system disks.

---

### 3.3 Third Mainline: Access Entry

Core Questions:

    Why can't HTTP 9000 be exposed directly?
    Why can HTTP be used internally?
    Why must HTTPS be used externally?
    Should API and Console be separated?
    What role does Nginx play in the architecture?

Stage Gains:

    9000 is MinIO S3 API port.
    9001 is MinIO Web Console port.
    Internal node communication can use HTTP in trusted networks.
    External client access must use HTTPS.
    API and Console should be separated.
    Nginx/LB should serve as a unified entry point.
    Console should not be exposed directly to the public internet.
    Backend 9000/9001 should be restricted by firewall or security group.
    In reverse proxy scenarios, attention should be paid to MINIO_SERVER_URL and MINIO_BROWSER_REDIRECT_URL.

---

### 3.4 Fourth Mainline: Permission Governance

Core Questions:

    Can root user be used for business?
    How to create regular users?
    How to design Policies?
    What to do if AccessKey leaks?

Stage Gains:

    Root user is only suitable for management and not for long-term business use.
    Business should use independent users and AccessKeys.
    Policies should adhere to minimal permissions.
    Bucket-level and Object-level permissions need separate design.
    Prefix-level permissions can restrict users to specific paths.
    DeleteObject permissions must be granted cautiously.
    AccessKey/SecretKey cannot be submitted to Git, cannot be written to frontend, cannot be shared among multiple users.
    After key leakage, prioritize disabling the user, investigating logs, then creating new users or keys.

---

### 3.5 Fifth Mainline: Data Protection

Core Questions:

    What is Erasure Coding?
    Can data still be read/written after node failure?
    How to recover after disk failure?
    Can Erasure Coding replace backups?

Stage Gains:

    MinIO uses Erasure Coding to protect object data.
    Erasure Coding can tolerate partial disk or node failures.
    Read quorum determines if data can be read.
    Write quorum determines if data can be written.
    After node failure, impact must be assessed through admin info, health, logs, and object read/write verification.
    After disk failure, mount points need to be recovered and healing observed.
    mc admin heal --dry-run is suitable for first checking repair scope.
    Erasure Coding cannot prevent accidental deletion, overwriting, malicious deletion, or full cluster failure.
    Erasure Coding is not a backup.

---

### 3.6 Sixth Mainline: Monitoring and Operations

Core Questions:

# What to Monitor in MinIO Production Environment Daily?
## How to Integrate with Prometheus?
## What Metrics Should Grafana Monitor?
## How to Design Alarms?

### Stage Harvest:

- `mc admin info` is the core command for manual inspections.
- `/minio/health/live` is used to check process liveness.
- `/minio/health/ready` is used to check if the service is ready.
- Docker logs can be used to view MinIO service logs.
- Nginx access.log / error.log can be used to troubleshoot entrypoint issues.
- Prometheus can collect MinIO metrics.
- Grafana should cover cluster, node, disk, bucket, S3 API, Nginx, certificate, and backup task metrics.
- Alarms should cover API unavailability, node offline, disk offline, capacity exceeding thresholds, 5xx errors, certificate expiration, backup failure, and healing taking too long to complete.
- Capacity management cannot rely solely on `df -h`; also monitor bucket, prefix, object count, and growth trends.

---

### 3.7 Seventh Mainline: Backup and Migration

Core Questions:

- How to back up MinIO?
- How to use `mc mirror`?
- How to perform cross-cluster migration?
- Why is `--remove` dangerous?
- How to choose between `mc mirror` and Bucket Replication?

### Stage Harvest:

- `mc mirror` is similar to rsync and can be used for synchronization between local directories, buckets, and MinIO clusters.
- Perform `--dry-run` before synchronization.
- `--overwrite` will overwrite objects with the same name in the target.
- `--remove` will delete objects in the target that no longer exist in the source, making it a high-risk parameter.
- `--watch` can be used for continuous synchronization, but it is not equivalent to a complete disaster recovery system.
- Assess capacity, object count, network, permissions, and rollback plans before migration.
- Verify capacity, object count, and critical object content after migration.
- Production backup tasks must have logs, alarms, and recovery drills.
- Long-term production-grade cross-cluster replication should evaluate Bucket Replication or Site Replication.

---

## Four, MinIO Core Architecture Convergence

### 4.1 MinIO Overall Access Model

MinIO's typical access model:

```
App / SDK / mc / aws cli
    |
    | HTTPS 443
    v
Nginx / LB Unified Entry Point
    |
    | HTTP 9000
    v
MinIO Distributed Cluster
    |
    v
Multi-node / Multi-disk / Erasure Coding
```

Core Understanding:

- Applications do not directly perceive backend nodes.
- Applications only access the unified S3 Endpoint.
- Backend nodes operate in a trusted internal network.
- External access must use HTTPS.
- Internal communication can use HTTP, but must have network boundaries.
- The console management entry must be separately protected.

---

### 4.2 Current Experimental Node Planning

This module plans to use the 10.0.0.0/24 network segment.

Known reserved addresses:

| IP | Purpose |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |

MinIO Experimental Addresses:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc Client / Test Client |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entry Point |

---

### 4.3 Image Version Convergence

MinIO server fixed version:

```
registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
```

Source image:

```
minio/minio:RELEASE.2025-04-22T22-12-26Z
```

mc client fixed version:

```
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
```

Source image:

```
minio/mc:RELEASE.2025-04-16T18-13-26Z
```

Reason for choosing fixed versions:

- Avoid inconsistencies in Web Console, command parameters, log formats, and experimental results caused by `latest` changes.
- Ensure reproducibility of experiments involving single-node Docker, distributed Docker, Nginx proxy, permissions, monitoring, backup, and migration.
- mc and MinIO server versions are close in time to reduce client-server capability differences.
- Production version selection still requires comprehensive evaluation of security patches, licenses, enterprise compliance, upgrade strategies, and official support.

Image source process:

1. First pull `minio/minio` and `minio/mc` from the official fixed version.
2. Retag to `registry.cn-hangzhou.aliyuncs.com/pub-syq`.
3. Push to the user's Alibaba Cloud image repository.
4. Subsequent notes will default to using the user's Alibaba Cloud image address to improve stability in domestic network environments.

---

## Five, MinIO vs Ceph RGW Comparison Summary

### 5.1 Commonalities

MinIO and Ceph RGW both belong to object storage capabilities.

Commonalities:

- Both are compatible with S3 API.
- Both use the Bucket / Object model.
- Both use AccessKey / SecretKey.
- Both are suitable for object storage scenarios such as images, attachments, backups, and archives.
- Both can provide HTTPS unified entry points via Nginx / LB.
- Both require permission governance, monitoring alarms, capacity management, and backup migration.

| Comparison Item | MinIO | Ceph RGW |
|---|---|---|
| System Focus | Specialized object storage | Object interface within Ceph's unified storage system |
| Underlying Mechanism | MinIO's own Erasure Coding | Ceph RADOS |
| Deployment Complexity | Relatively low | Depends on a complete Ceph cluster |
| Operation Complexity | Lower than Ceph | High |
| Storage Type | Primarily object storage | Supports RBD, CephFS, RGW simultaneously |
| Common Access Points | S3 API / Console | S3 / Swift API |
| Suitable Scenarios | Private S3, lightweight object storage, application attachments and backups | Existing Ceph cluster needing object interface |
| Learning Focus | S3, EC, mc, Nginx, permissions, backup | RADOS, Pool, PG, CRUSH, RGW |

---

### 5.3 Advantages of Learning MinIO After Ceph

Having already learned Ceph, MinIO becomes easier to understand:

    Already understands object storage.
    Already understands Bucket and Object.
    Already understands AccessKey / SecretKey.
    Already understands HTTPS unified entry point.
    Already understands the importance of monitoring, alerts, backup, and recovery.
    Already understands "replication/erasure coding is not backup".
    Already has awareness of distributed storage fault domains, capacity, and recovery.

MinIO phase focus shifts from "what is object storage" to:

    MinIO's own deployment model.
    MinIO Erasure Coding.
    mc toolset.
    Nginx unified entry point.
    Users and Policy.
    Prometheus monitoring.
    mc mirror backup migration.
    Production readiness validation.

---

## SixI don't know.Capability Evolution from "Deployment" to "Production Operations"

### 6.1Primary Stage: Can Start

Primary stage focuses on:

    How to write Docker run?
    How to access 9000?
    How to login to 9001?
    How to create Bucket?
    How to upload/download objects?

This is entry-level capability.

---

### 6.2 Operation Stage: Can Troubleshoot

Operation stage focuses on:

    How to troubleshoot 9000 connectivity issues?
    How to troubleshoot Console login failures?
    How to troubleshoot mc alias failures?
    How to troubleshoot large file upload failures?
    How to troubleshoot 502 / 413 / 504 errors?
    How to troubleshoot AccessDenied?
    How to troubleshoot SignatureDoesNotMatch?
    How to troubleshoot node offline?
    How to troubleshoot disk offline?
    How to troubleshoot Bucket capacity spikes?

This is hands-on capability.

---

### 6.3 Advanced SRE Stage: Can Govern

Advanced SRE stage focuses on:

    Is the architecture highly available?
    Are there single points in the entry?
    Is external access fully HTTPS?
    Is internal HTTP secured with boundary?
    Is Console access source restricted?
    Is root user deployed for business?
    Is Policy minimal permissions?
    Can AccessKey be rotated?
    Is Erasure Coding mistakenly considered backup?
    Is cross-cluster backup implemented?
    Is recovery drill conducted?
    Is Prometheus covering key metrics?
    Can alerts notify specific people?
    Is capacity threshold and expansion plan in place?
    Are high-risk operations subject to approval and review?

This is production governance capability.

---

## SevenI don't know.MinIO Core Command Review

### 7.1 Docker Deployment and Inspection

    docker ps | grep minio
    docker ps -a | grep minio
    docker logs --tail=100 minio
    docker logs -f minio
    docker inspect minio
    docker stats minio

---

### 7.2 Health Check

    curl -I http://10.0.0.41:9000/minio/health/live
    curl -I http://10.0.0.41:9000/minio/health/ready

    curl -k -I https://s3.minio.local/minio/health/live
    curl -k -I https://s3.minio.local/minio/health/ready

---

### 7.3 mc Basic Commands

    mc alias set
    mc alias list
    mc admin info
    mc mb
    mc rb
    mc ls
    mc cp
    mc stat
    mc cat
    mc du
    mc find
    mc rm
    mc mirror

---

### 7.4 User and Permissions Commands

    mc admin user add
    mc admin user list
    mc admin user info
    mc admin user disable
    mc admin user enable
    mc admin user remove

    mc admin policy create
    mc admin policy list
    mc admin policy info
    mc admin policy attach
    mc admin policy detach
    mc admin policy remove

---

### 7.5 Data Protection and Recovery Commands

    mc admin info
    mc admin heal --recursive --dry-run
    mc admin heal --recursive

---

### 7.6 Monitoring Related Commands

    mc admin prometheus generate
    curl -H "Authorization: Bearer <token>" https://s3.minio.local/minio/v2/metrics/cluster

---

### 7.7 Backup and Migration Commands

    mc mirror --dry-run
    mc mirror --summary
    mc mirror --overwrite
    mc mirror --remove
    mc mirror --watch
    mc mirror --include
    mc mirror --exclude
    mc mirror --limit-upload
    mc mirror --limit-download
    mc mirror --max-workers

---

## EightI don't know.MinIO Troubleshooting Path Summary /think

| Fault Phenomenon | First Entry | Second Entry | Deep Direction |
|---|---|---|---|
| API Not Working | curl health ready | Nginx error.log | Backend, Port, Firewall |
| Console Not Working | curl console domain | Nginx console configuration | 9001, Redirect, WebSocket |
| mc alias Failed | Check endpoint | AccessKey / SecretKey | Certificate, 9000, Permissions |
| Upload Failed | mc cp small file test | Nginx error.log | Permissions, Capacity, Write Quorum |
| Large File Upload Failed | Nginx error.log | client_max_body_size | Proxy Timeout, Buffer |
| 502 | Nginx error.log | Backend health | Upstream, Backend Crash |
| 413 | Nginx Configuration | client_max_body_size | Upload Limit |
| 504 | Nginx Timeout | Backend Performance | Network, Disk, Timeout |
| AccessDenied | User Info | Policy Info | Action, Resource, Prefix |
| SignatureDoesNotMatch | Host Header | Endpoint | HTTP/HTTPS, Time, Signature |
| Node Offline | mc admin info | docker ps/logs | Host, Docker, Network |
| Disk Offline | mc admin info | df/lsblk/dmesg | Mount, Hardware, Permissions |
| Bucket Capacity Surge | mc du | mc find | Business, Abnormal Upload, Lifecycle |
| Backup Failed | Mirror Log | Alias / Permissions | Network, Target Capacity, Permissions |

---

## Nine. MinIO Production Risk Summary

### 9.1 Data Risk

Common Risks:

- Accidental Bucket Deletion
- Accidental Prefix Deletion
- Accidental Object Deletion
- Using mc rm --recursive --force
- Using mc mirror --remove
- Application Overwriting Objects
- AccessKey Leak Leading to Malicious Deletion
- Mistaking Erasure Coding for Backup
- Backup Without Recovery Drill

Governance Methods:

    Backup Critical Buckets.
    Approval for High-Risk Deletion Operations.
    Policy Minimum Permissions.
    Caution in Granting DeleteObject.
    Enable Backup Task Logs.
    Regular Recovery Drills.
    Evaluate Version Control or Replication When Necessary.

---

### 9.2 Security Risk

Common Risks:

- Root User for Business Use
- AccessKey / SecretKey Submitted to Git
- HTTP 9000 Exposed to Public Internet
- Console 9001 Exposed to Public Internet
- Console No Source Restriction
- Business User Overprivileged
- Policy Using s3:* or Resource:*
- Multiple Businesses Sharing a Key Group
- Key Leak Without Rotation

Governance Methods:

    Root User Only for Management.
    Business Use Independent Users.
    Policy Minimum Permissions.
    External Forced HTTPS.
    Console Source Restriction.
    Keys Not Stored in Repository.
    Keys Can Be Rotated.
    Disable User Immediately After Leak.

---

### 9.3 Capacity Risk

Common Risks:

- Disk Near Full
- Bucket Abnormal Growth
- Too Many Small Objects
- Temporary Files Not Cleaned
- Backup Files Overwritten
- Log Archive No Lifecycle
- Only Check df, Not Bucket
- Expansion Plan Delayed

Governance Methods:

    Establish Bucket Ownership.
    Daily Capacity Inspection.
    Monitor Bucket Growth Trends.
    Set Capacity Threshold Alerts.
    Evaluate Lifecycle Policies.
    Plan Expansion in Advance.
    Backup Data Separate Governance.

---

### 9.4 Operations Risk

Common Risks:

- No Monitoring
- No Alerts
- No Log Collection
- Nginx Single Point
- Certificate Expired
- Entry Configuration Unmaintained
- High-Risk Operations No Approval
- No Fault Review
- Backup Task Failure Unnoticed

Governance Methods:

    Prometheus Monitoring.
    Grafana Dashboard.
    Alertmanager Alerts.
    Nginx High Availability.
    Certificate Expiry Monitoring.
    Backup Task Alerts.
    High-Risk Operations Review.
    Fault Handling Templated.

---

## Ten. Production-Ready MinIO Cluster Acceptance Checklist

### 10.1 Architecture Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Deployment Mode | Multi-Node Multi-Disk |  |
| Number of Nodes | At Least 4 Nodes or Per Business Planning |  |
| Data Disk | Independent Data Disk, Not Overwrite System Disk |  |
| Image Version | Fixed Version, Not Use latest |  |
| Entry Layer | Nginx / LB Unified Entry |  |
| External Protocol | HTTPS |  |
| Internal Communication | HTTP Acceptable, But Must Be Trusted Intranet |  |
| Console | Independent Entry with Source Restriction |  |
| Entry High Availability | Production Should Avoid Single Nginx |  |

---

### 10.2 Function Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Bucket Creation | Normal |  |
| Object Upload | Small File, Large File Both Normal |  |
| Object Download | Normal |  |
| mc alias | Can Connect via HTTPS Entry |  |
| admin info | Normal Display of Nodes and Disks |  |
| health live | Normal |  |
| health ready | Normal |  |
| Console | Can Log In and Access Controlled |  |

---

### 10.3 Permission Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Root User | Not Distributed to Business |  |
| Business User | Independent User |  |
| Policy | Minimum Permissions |  |
| DeleteObject | Caution in Granting |  |
| Bucket Permissions | Isolated by Business |  |
| Prefix Permissions | Restrict When Necessary |  |
| Key Management | Not Submitted to Git, Not Explicitly Spread |  |
| Key Rotation | Has Process |  |

---

### 10.4 Data Protection Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Erasure Coding | Understood and validated |  |
| Node Failure Drill | Executed |  |
| Disk Failure Drill | Executed |  |
| Healing Check | Validated |  |
| Backup Strategy | Designed |  |
| mc mirror | Tested |  |
| Recovery Drill | Executed |  |
| Data Recovery from Accidental Deletion | Has a plan |  |

---

### 10.5 Monitoring and Alert Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Prometheus | Has collected MinIO metrics |  |
| Grafana | Has cluster and capacity dashboards |  |
| API Alert | ready / 5xx / latency |  |
| Node Alert | offline |  |
| Disk Alert | offline / capacity |  |
| Bucket Alert | Abnormal capacity growth |  |
| Nginx Alert | 502 / 504 / 413 |  |
| Certificate Alert | Warning before expiration |  |
| Backup Alert | Failure notification |  |

---

## ElevenI don't know.MinIO Subsequent Strengthening Directions

### 11.1 MinIO Advanced Directions

Subsequent areas to explore:

- Bucket Versioning
- Object Lock
- Lifecycle Policy
- Bucket Replication
- Site Replication
- Batch Framework
- ILM / Tiering
- KMS / SSE Encryption
- OIDC / LDAP / AD Integration
- Multi-tenant Object Storage Governance
- Large-scale Bucket Performance Optimization
- Small Object Governance
- Online Expansion Methods
- Production Upgrade and Rollback

---

### 11.2 Platformization Directions

Subsequent areas to explore:

- Self-service Object Storage Application Platform
- Bucket Lifecycle Management Platform
- Auto-generation and Rotation of AccessKey
- Bucket Capacity Quota
- Bucket CMDB Association
- Object Storage Audit Platform
- Backup Task Platformization
- Abnormal Deletion Detection
- S3 API Usage Statistics
- Unified Management of Multi-cluster MinIO
- Integration with DevOps Artifact Management
- Integration with AI Dataset Storage

---

### 11.3 Relationship with Subsequent RustFS Learning

After mastering MinIO, learning RustFS will be easier because the following concepts are already understood:

    Object Storage Model.
    S3 API.
    Bucket / Object / Prefix.
    Single-node and Cluster Mode.
    Reverse Proxy and HTTPS.
    AccessKey / SecretKey.
    Permission Governance.
    Monitoring and Logs.
    Backup and Migration.
    Internal HTTP and External HTTPS Entry Design.

RustFS phase should focus on comparisons:

    Compatibility between RustFS and MinIO.
    Deployment methods of RustFS.
    Cluster capabilities of RustFS.
    Production maturity of RustFS.
    Monitoring and backup capabilities of RustFS.
    Whether RustFS is suitable to replace or complement MinIO.

---

### 11.4 Relationship with Longhorn Learning

MinIO is an object storage system.

Longhorn is a Kubernetes-native block storage.

Comparison:

| Comparison Item | MinIO | Longhorn |
|---|---|---|
| Type | Object Storage | Block Storage |
| Access Method | S3 API | Kubernetes PVC |
| Typical Ports | 9000 / 9001 | Kubernetes internal components |
| Data Model | Bucket / Object | Volume / Replica |
| Usage Objects | Applications, SDKs, backup tools | Pods, StatefulSets |
| Key Capabilities | S3, objects, backup, permissions | PVC, replicas, volume recovery, CSI |

After learning MinIO, learning Longhorn requires a shift in mindset:

    MinIO focuses on objects and S3.
    Longhorn focuses on volumes, replicas, nodes, and Kubernetes CSI.
    MinIO is suitable for attachments, images, and archives.
    Longhorn is suitable for stateful application volumes.

---

## TwelveI don't know.Interview Expression Summary

If asked in an interview:

    How familiar are you with MinIO?

You can respond:

I have systematically studied and organized the complete MinIO ecosystem from fundamentals to production operations, far beyond just docker run to start a single-node service.  
I understand MinIO as an object storage system compatible with the S3 API, with core models including Bucket, Object, Prefix, AccessKey, and SecretKey. It differs from block storage and file storage, with applications typically uploading/downloading objects via S3 API, mc, aws cli, or SDKs.  
At the deployment level, I can distinguish between single-node single-disk, single-node multi-disk, and multi-node multi-disk deployment modes. Single-node is suitable for learning, single-node multi-disk can demonstrate Erasure Coding concepts, but production environments recommend multi-node multi-disk. I have planned 4-node multi-disk distributed MinIO clusters, knowing that endpoint lists must be consistent across nodes, data directories must use dedicated data disks, and never write to system disks.  
For entry design, I separate internal communication from external access. MinIO nodes or Nginx to backend nodes can use HTTP in trusted internal networks, but external clients must access via HTTPS. API and Console endpoints should be separated, e.g., s3.example.com proxies 9000, minio-console.example.com proxies 9001, with Console access restricted to maintenance network segments.  
For permissions, I never grant root users to business applications, instead creating independent users and Policies for each business or environment. Policies implement minimal permissions by Bucket or Prefix, with separate read/write/delete controls. AccessKey/SecretKey are managed by key level, never committed to Git, and after leakage, disable users first, then investigate logs and rotate keys.  
For data protection, I understand MinIO uses Erasure Coding to tolerate partial disk/node failures, and grasp the basic meaning of read quorum and write quorum. When nodes or disks fail, I troubleshoot via mc admin info, health interface, Docker logs, disk mounts, and Nginx logs, and observe repair scope via mc admin heal --dry-run. However, I also know Erasure Coding is not backup and cannot prevent accidental deletion or full cluster failures.  
For production operations, I integrate Prometheus and Grafana, monitoring nodes, disks, Bucket capacity, object counts, S3 request volumes, 4xx/5xx errors, API latency, Nginx entry points, certificate expiration, and backup tasks. For backups and migrations, I use mc mirror for scheduled backups or cross-cluster migrations, perform dry-run before execution, use --overwrite and --remove cautiously, and validate capacity, object counts, and critical object contents post-migration.  
Overall, my understanding of MinIO goes beyond installation, covering deployment, entry design, security, permissions, monitoring, fault handling, backup, and production readiness from an advanced SRE perspective.

---

## Thirteen. Summary of This Section

This article completes the MinIO phase summary:

1. MinIO is an object storage system compatible with the S3 API.  
2. MinIO is suitable for non-structured data like images, attachments, backups, archives, logs, and artifacts.  
3. The core model of MinIO is Bucket, Object, and Prefix.  
4. 9000 is the S3 API port, and 9001 is the Web Console port.  
5. Single-node single-disk is suitable for learning but not for production.  
6. Single-node multi-disk can demonstrate Erasure Coding concepts but cannot resolve node failures.  
7. Multi-node multi-disk is closer to production deployment.  
8. External client access must use HTTPS.  
9. Internal trusted networks can use HTTP, but access boundaries must be restricted.  
10. Nginx / LB should provide unified entry points.  
11. API and Console should be separated.  
12. Console should not be exposed directly to the public internet.  
13. Root users should not be given to business applications.  
14. Business users should use independent AccessKey and minimal permission Policies.  
15. Erasure Coding can tolerate partial disk or node failures.  
16. Erasure Coding cannot replace backups.  
17. mc is an essential tool for MinIO operations.  
18. mc admin info is the core command for inspections.  
19. mc mirror can be used for backups and migrations, but --remove is a high-risk parameter.  
20. Prometheus, Grafana, and Alertmanager are core components for production monitoring and alerts.  
21. Capacity management must consider clusters, nodes, disks, Buckets, Prefixes, and object counts simultaneously.  
22. Production MinIO must have monitoring, alerts, backups, recovery drills, security governance, and high-risk operation processes.  
23. After studying MinIO, understanding RustFS and other S3-compatible object storages becomes easier.  
24. At this point, the MinIO phase has completed the full closed-loop from S3 object storage to production-ready clusters.

---

## Fourteen. Reference Documents

MinIO Official Documentation:  

    https://min.io/docs/minio/linux/index.html  

MinIO Docker Deployment Documentation:  

    https://min.io/docs/minio/container/index.html  

MinIO Multi-Node Multi-Disk Deployment Documentation:  

    https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html  

MinIO Erasure Coding Documentation:  

    https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html  

MinIO Nginx Reverse Proxy Documentation:  

    https://min.io/docs/minio/linux/integrations/setup-nginx-proxy-with-minio.html  

MinIO mc Client Documentation:  

    https://min.io/docs/minio/linux/reference/minio-mc.html  

MinIO mc Admin Documentation:  

    https://min.io/docs/minio/linux/reference/minio-mc-admin.html  

MinIO User and Permissions Documentation:  

    https://min.io/docs/minio/linux/administration/identity-access-management.html  

MinIO Prometheus Monitoring Documentation:  

    https://min.io/docs/minio/linux/operations/monitoring/collect-minio-metrics-using-prometheus.html  

MinIO mc mirror Documentation:  

    https://min.io/docs/minio/linux/reference/minio-mc/mc-mirror.html  

MinIO Bucket Replication Documentation:  

    https://min.io/docs/minio/linux/administration/bucket-replication.html  

MinIO Site Replication Documentation:  

    https://min.io/docs/minio/linux/administration/site-replication.html  

AWS S3 API Documentation:  

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html  

Nginx Official Documentation:  

    https://nginx.org/en/docs/  

Prometheus Official Documentation:  

    https://prometheus.io/docs/introduction/overview/  

Grafana Official Documentation:  

    https://grafana.com/docs/