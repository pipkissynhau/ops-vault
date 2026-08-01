# RustFS Directory Index

Recommended path: 05-Storage/04-RustFS/00-RustFS Directory Index.md

Tags: #RustFS #ObjectStorage #S3 #MinioReplacement #DistributedStorage #Docker #ReverseAgent #HTTPS #mc #AWSCLI #AdvancedSre #ProductionTransport

---

## I. Module Description

RustFS is an S3-compatible object storage system written in Rust.

Its learning positioning is:

    As a supplementary module to MinIO object storage
    Focus on understanding deployment, usage, operation, and production boundaries of new S3-compatible object storage
    Experiment through single-node Docker and multi-node Docker/VM cluster methods
    Focus on S3 API, Bucket, Object, AccessKey, SecretKey, HTTPS entry, reverse proxy, logs, capacity, fault recovery, and ecosystem compatibility

Like MinIO, RustFS belongs to the object storage direction.

It is NOT:

    Block storage
    File system storage
    Kubernetes CSI block storage
    Longhorn alternative
    Ceph RBD alternative
    Ordinary NFS alternative

It primarily addresses:

    Applications uploading and downloading objects via S3 API
    Private object storage service
    Storage of unstructured data such as images, attachments, backup packages, log archives, AI datasets, model files
    Replacement, compatibility, migration, and comparative study of MinIO-like object storage systems

---

## II. Module Learning Objectives

After completing the RustFS module, you should be able to:

1. Understand RustFS's basic positioning.
2. Understand the relationship between RustFS and the S3 object storage model.
3. Understand similarities and differences between RustFS and MinIO.
4. Deploy a single-node RustFS service using Docker.
5. Plan RustFS data directories, access ports, and access entry points.
6. Access RustFS via S3 clients.
7. Use mc or AWS CLI to create Buckets, upload objects, download objects, and verify access.
8. Understand node, disk, and access entry planning for RustFS cluster deployment.
9. Design internal HTTP communication and external HTTPS access methods.
10. Use Nginx to reverse proxy RustFS API entry points.
11. Understand AccessKey/SecretKey, Bucket permissions, and access control.
12. Configure or plan HTTPS, certificates, and unified entry points.
13. View logs, capacity, and node status.
14. Troubleshoot issues like RustFS service startup failures, port conflicts, permission anomalies, and S3 access failures.
15. Understand the production applicability boundaries of RustFS as a new object storage solution.
16. Judge whether RustFS is suitable for testing, replacement validation, or production deployment.
17. Compare RustFS with MinIO, Ceph RGW, and cloud vendor OSS/S3.
18. Establish an advanced SRE operations methodology for the RustFS module.

---

## III. Experiment Implementation Methods

The RustFS module experiment methods are:

    Single-node Docker deployment
    Multi-node Docker/VM cluster deployment

No dependency on Kubernetes.

No use of Longhorn.

No modification to containerd.

No disruption to KubernetesBottom runtime.

This module still uses the 10.0.0.0/24 experimental network segment.

Occupied or reserved addresses:

| IP | Purpose |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor / operations services |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |
| 10.0.0.41 - 10.0.0.46 | MinIO experimental nodes and entry planning |

RustFS recommends independent address planning to avoid mixing with Ceph, MinIO, and Longhorn experiments.

RustFS experiment planning:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Node 1 |
| 10.0.0.52 | rustfs-node02 | RustFS Node 2 |
| 10.0.0.53 | rustfs-node03 | RustFS Node 3 |
| 10.0.0.54 | rustfs-node04 | RustFS Node 4 |
| 10.0.0.55 | rustfs-client | S3 Client / mc / aws cli test node |
| 10.0.0.56 | rustfs-entry | Nginx HTTPS unified entry |

---

## IV. Operating System and Base Environment

### 4.1 Default Experimental System

Default use:

    Ubuntu Server 22.04.5 LTS

Reasons:

    Consistent with previous Kubernetes, MinIO, and Longhorn experimental systems.
    Simple Docker installation.
    Unified network, disk, Nginx, and certificate configuration methods.
    Facilitates futureDeposition into reusable production operations notes.

---

### 4.2 Optional Systems

Optional:

    Rocky Linux 9

Rocky Linux 9 is mainly used to supplement enterprise production scenarios of common RHEL-based distributions.

The RustFS module's subsequent deployment notes should cover:

    Ubuntu 22.04 Docker installation method
    Rocky Linux 9 Docker installation method
    Firewall handling
    SELinux considerations
    Data directory permissions
    systemd / Docker restart policies
    Nginx reverse proxy

---

## V. Relationship Between RustFS and Previous Storage Modules

### 5.1 RustFS and MinIO

RustFS and MinIO both belong to:

    S3-compatible object storage

Similarities:

    Both provide S3 API.
    Both use Bucket/Object model.
    Both can be accessed via AccessKey/SecretKey.
    Both can be operated using mc or AWS CLI.
    Both are suitable for object storage scenarios such as images, attachments, backup packages, archives, and AI datasets.
    Both can be accessed via a unified entry point behind Nginx/LB.
    Both require HTTPS, permission governance, backup migration, logs, and capacity management.

Differences:

| Comparison Item | MinIO | RustFS |
|---|---|---|
| Maturity | More mature, with more production cases | New solution, requires careful evaluation |
| Technical Implementation | Go | Rust |
| Learning Focus | Object storage core module | MinIO successor and comparison module |
| Ecosystem Compatibility | Broad ecosystem | Needs focused compatibility verification |
| Operations Method | Relatively mature | Needs experimental validation |
| Production Recommendation | Can be evaluated as a mature private object storage solution | Needs assessment based on version, features, stability, and community activity |

Learning order:

    Learn MinIO first
    Then learn RustFS

This makes it easier to understand:

    S3 API
    Bucket / Object
    AccessKey / SecretKey
    Reverse Proxy
    HTTPS
    mc Tool
    Access Control
    Backup Migration
    Object Storage Production Boundaries

---

### 5.2 RustFS vs Ceph RGW

RustFS and Ceph RGW can both provide object storage capabilities.

Comparison:

| Comparison Item | RustFS | Ceph RGW |
|---|---|---|
| Type | Independent S3-compatible object storage | Ceph unified storage system's object gateway |
| Underlying Architecture | RustFS own object storage engine | Ceph RADOS |
| Deployment Complexity | Relatively lightweight | Higher |
| Operations Complexity | Moderate, requires validation | High |
| Storage Type | Object storage | Unified object, block, and file system |
| Use Cases | Private S3, lightweight object storage, replacement verification | Existing Ceph cluster needing object interface |
| Learning Focus | S3 compatibility, deployment, entry point, permissions, compatibility | RADOS, Pool, PG, CRUSH, RGW |

One sentence:

    RustFS is a lighter S3 object storage direction.
    Ceph RGW is the object interface within Ceph's large unified storage system.

---

### 5.3 RustFS vs Longhorn

RustFS and Longhorn are fundamentally different.

| Comparison Item | RustFS | Longhorn |
|---|---|---|
| Type | Object storage | Kubernetes block storage |
| Access Method | S3 API | PV / PVC / CSI |
| Usage Objects | Applications, SDKs, mc, aws cli | Pods, Deployments, StatefulSets |
| Data Unit | Bucket / Object | Volume / Replica |
| Typical Scenarios | Images, attachments, backups, AI datasets | Database data disks, application persistent volumes |
| Direct Mount as Directory | No | Yes |
| K8s CSI Membership | No | Yes |

One sentence:

    Pods need a persistent disk, use Longhorn.
    Applications needing object upload/download interface, use RustFS or MinIO.

---

## Six. Module Directory Explanation

Current RustFS module directory:

    00-RustFS Directory Index
    01-RustFS Foundation: S3-compatible object storage and use cases
    02-RustFS Deployment Modes: Single-node and cluster mode understanding
    03-RustFS Single-node Deployment Practice: Service startup, data directory, and access verification
    04-RustFS Cluster Deployment Practice: Multi-node, multi-disk, and access entry
    05-RustFS vs MinIO: Architecture, deployment, ecosystem, and operations differences
    06-RustFS Client Access: S3 API, mc tool, and application integration
    07-RustFS Permissions and Security: Access keys, HTTPS, and reverse proxy
    08-RustFS Operations and Troubleshooting: Logs, capacity, node anomalies, and recovery
    99-RustFS Stage Summary: Positioning and practice boundaries of new object storage

---

## Seven. Notes Objectives

### 00-RustFS Directory Index

File:

    00-RustFS Directory Index.md

Objective:

    Overview RustFS module learning path.
    Clarify RustFS's relationship with MinIO, Ceph RGW, and Longhorn.
    Clarify experiment environment, node planning, access entry, and learning boundaries.
    Establish advanced SRE learning methodology for RustFS modules.

---

### 01-RustFS Foundation: S3-compatible object storage and use cases

File:

    01-RustFS Foundation: S3-compatible object storage and use cases.md

Focus:

    What is RustFS.
    What is S3-compatible object storage.
    What are Bucket / Object / Prefix.
    What scenarios is RustFS suitable for.
    What scenarios is RustFS unsuitable for.
    Basic relationship between RustFS and MinIO.
    Why RustFS's production maturity needs cautious evaluation.

Hands-on Objective:

    Understand S3 Endpoint.
    Understand AccessKey / SecretKey.
    Understand Bucket.
    Understand Object.
    Be able to explain what capabilities RustFS is suitable for verification.

---

### 02-RustFS Deployment Modes: Single-node and cluster mode understanding

File:

    02-RustFS Deployment Modes: Single-node and cluster mode understanding.md

Focus:

    What single-node mode is suitable for.
    What cluster mode is suitable for.
    How to plan data directories.
    How to plan node numbers.
    How to plan ports.
    How to differentiate internal communication and external access.
    Why learn single-node first, then multi-node clusters.
    Why experimental deployment cannot be directly equated with production deployment.

Hands-on Objective:

    Plan single-node RustFS.
    Plan multi-node RustFS.
    Plan reverse proxy entry.
    Plan data disk directories.
    Plan access domain names and certificates.

---

### 03-RustFS Single-node Deployment Practice: Service startup, data directory, and access verification

File:

    03-RustFS Single-node Deployment Practice: Service startup, data directory, and access verification.md

Focus:

    Docker single-node deployment of RustFS.
    Data directory preparation.
    Port mapping.
    Environment variables.
    Service startup.
    Log viewing.
    API access verification.
    Using mc or AWS CLI to create Bucket.
    Upload/download objects.
    Data persistence after service restart.

Hands-on Objective:

    Be able to start RustFS single-node service on 10.0.0.51.
    Be able to create data directories.
    Be able to run RustFS via Docker.
    Be able to view container logs.
    Be able to access via S3 client.
    Be able to create Bucket.
    Be able to upload and download objects.
    Be able to verify data persistence.

---

### 04-RustFS Cluster Deployment Practice: Multi-node, multi-disk, and access entry

File:

    04-RustFS Cluster Deployment Practice: Multi-node, multi-disk, and access entry.md

Focus:

Multi-node RustFS Cluster Deployment  
Multi-disk Data Directory Planning  
Node-to-node Access  
Port Opening  
Cluster Startup Order  
Node Failure Observation  
Unified Entry Design  
Nginx / LB Reverse Proxy  
Internal HTTP and External HTTPS  
Data Recovery and Node Recovery Basics  

Hands-on Objectives:  

    Can plan a 4-node RustFS cluster.  
    Can prepare data directories for each node.  
    Can start multiple RustFS nodes via Docker.  
    Can verify API access for each node.  
    Can access via a unified entry point.  
    Can observe the impact of node anomalies.  
    Can understand production boundaries of cluster mode.  

---

### 05-RustFS vs MinIO: Architecture, Deployment, Ecosystem, and Operations Differences  

Files:  

    05-RustFS vs MinIO: Architecture, Deployment, Ecosystem, and Operations Differences.md  

Key Points:  

    RustFS and MinIO are both S3-compatible object storage systems.  
    Similar architecture points.  
    Deployment differences.  
    Client compatibility differences.  
    Ecosystem maturity differences.  
    Operations tool differences.  
    Comparison of monitoring, logging, permissions, and backup capabilities.  
    How to judge during production selection.  
    Why RustFS is more suitable as a new solution for verification rather than blind replacement.  

Hands-on Objectives:  

    Use the same mc client to access both MinIO and RustFS.  
    Create identical Buckets.  
    Upload identical objects.  
    Verify S3 basic compatibility.  
    Compare logs, ports, permissions, consoles, and operations methods.  
    Output a selection judgment table.  

---

### 06-RustFS Client Access: S3 API, mc Tool, and Application Integration  

Files:  

    06-RustFS Client Access: S3 API, mc Tool, and Application Integration.md  

Key Points:  

    S3 API access methods.  
    mc tool configuration.  
    AWS CLI configuration.  
    Bucket creation.  
    Object upload/download.  
    Presigned URL.  
    Application integration Endpoint.  
    HTTP/HTTPS access differences.  
    Understanding Path-style and Virtual-hosted-style access.  
    Common client compatibility issues.  

Hands-on Objectives:  

    Configure RustFS using mc alias set.  
    Create a Bucket using mc mb.  
    Upload objects using mc cp.  
    List objects using mc ls.  
    Access RustFS using AWS CLI.  
    Verify how applications fill in Endpoint, AccessKey, SecretKey, and Bucket.  
    Troubleshoot SignatureDoesNotMatch, AccessDenied, and Endpoint errors.  

---

### 07-RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxy  

Files:  

    07-RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxy.md  

Key Points:  

    Differentiation between root account and business accounts.  
    AccessKey/SecretKey management.  
    Bucket permission control.  
    HTTPS entry point.  
    Nginx reverse proxy.  
    Unified domain entry.  
    Internal HTTP and external HTTPS design.  
    Whether to expose the 9000 API port.  
    Certificate configuration.  
    Firewall and source restriction.  
    Log auditing.  
    Key leakage handling.  

Hands-on Objectives:  

    Configure Nginx HTTPS entry point.  
    Use internal HTTP for backend RustFS.  
    Use HTTPS for external clients.  
    Access using certificates.  
    Verify mc usage of HTTPS Endpoint.  
    Restrict access sources for management entry.  
    Record key rotation and leakage handling procedures.  

---

### 08-RustFS Operations and Troubleshooting: Logs, Capacity, Node Anomalies, and Recovery  

Files:  

    08-RustFS Operations and Troubleshooting: Logs, Capacity, Node Anomalies, and Recovery.md  

Key Points:  

    Service log viewing.  
    Docker container status.  
    Data directory capacity.  
    Node disk anomalies.  
    Port anomalies.  
    S3 access failure.  
    AccessDenied.  
    SignatureDoesNotMatch.  
    Nginx 502 / 413 / 504.  
    Node anomaly recovery.  
    Risk of data directory accidental deletion.  
    Backup migration approach.  
    Prometheus / log collection reservation.  

Hands-on Objectives:  

    Can troubleshoot RustFS container startup failure.  
    Can troubleshoot port conflicts.  
    Can troubleshoot data directory permissions.  
    Can troubleshoot client access failure.  
    Can troubleshoot Nginx reverse proxy errors.  
    Can troubleshoot Bucket/Object access anomalies.  
    Can establish daily inspection commands.  
    Can establish production fault record templates.  

---

### 99-RustFS Stage Summary: Positioning and Practice Boundaries of New Object Storage  

Files:  

    99-RustFS Stage Summary: Positioning and Practice Boundaries of New Object Storage.md  

Key Points:  

    Summarize RustFS positioning.  
    Summarize RustFS's relationship with MinIO.  
    Summarize RustFS's deployment, access, security, operations, and production boundaries.  
    Summarize whether RustFS is suitable for production.  
    Summarize RustFS's value for advanced SRE learning.  
    Summarize interview expressions.  
    Establish a selection methodology for new object storage.  

---

## VIII. RustFS Experiment Topology Planning  

### 8.1 Single-machine Mode Topology  

Single-machine Docker experiment: /think

```
┌───────────────────────────────┐
│ rustfs-node01                 │
│ 10.0.0.51                     │
│                               │
│ Docker                        │
│   └── rustfs                  │
│      ├── API Port             │
│      └── Data: /data/rustfs   │
└───────────────────────────────┘
                  ^
                  |
                  | S3 API
                  |
┌───────────────────────────────┐
│ rustfs-client                 │
│ 10.0.0.55                     │
│ mc / aws cli                  │
└───────────────────────────────┘

Single-node mode is used for:

    Service startup verification
    S3 API compatibility verification
    Bucket / Object basic operations
    Data persistence verification
    Client access verification

Not used for:

    Production high availability
    Node failure recovery
    Multi-node disaster recovery verification

---

### 8.2 Multi-node Mode Topology

Multi-node Docker / VM cluster experiment:

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
    │ Nginx HTTPS unified entry     │
    └───────────────────────────────┘

    ┌───────────────────────────────┐
    │ rustfs-client 10.0.0.55       │
    │ mc / aws cli / app            │
    └───────────────────────────────┘

---

### 8.3 Access Entry Design

Recommended entry strategy:

    Internal node communication: HTTP
    External client access: HTTPS
    Unified entry: Nginx / LB
    Clients only access unified domain name
    Backend node ports not directly exposed to public internet

Example:

    rustfs-api.example.com -> Nginx HTTPS -> RustFS API backend nodes

Experimental domain names can use:

    rustfs.local
    s3.rustfs.local

Local hosts example:

    10.0.0.56 s3.rustfs.local

---

## IX. Port Planning

Specific ports are determined by the official RustFS documentation and actual startup parameters.

This module's planning principles:

| Port Type | Description | Exposure Recommendation |
|---|---|---|
| RustFS API Port | S3 API access port | Expose internally, external via Nginx |
| RustFS Console / Management Port | If current version provides management interface | Restrict source, not directly public |
| Nginx 443 | External HTTPS unified entry | Open to clients |
| Ngin, 80 | Can be used for HTTP redirect | Optional |
| SSH 22 | Node operations | Only on maintenance network segment |

Production principles:

    API ports not directly exposed to public internet.
    Management ports not directly exposed to public internet.
    External access must use HTTPS.
    Internal HTTP only for trusted networks.
    Firewall source restrictions.
    Port planning documented to avoid conflicts.

---

## X. Data Directory Planning

Single-node mode:

    /data/rustfs

Multi-node mode:

    /data/rustfs

For multiple disks, can plan:

    /data/rustfs/disk1
    /data/rustfs/disk2
    /data/rustfs/disk3
    /data/rustfs/disk4

Data disk recommendations:

    Use independent data disks.
    Do not use system disk for production data.
    Use xfs or ext4.
    Mount with /etc/fstab.
    Mount parameters can use noatime.
    Each node capacity should be as consistent as possible.
    Disk performance should be as consistent as possible.

Check commands:

    df -hT
    lsblk -f
    mount | grep /data
    du -sh /data/rustfs

High-risk warning: /think

Do not manually delete RustFS data directories.
Do not directly rm -rf /data/rustfs.
Do not mix data directories with other business logs or caches.
Permission anomalies in data directories will cause service startup failures or object write failures.

---

## 11. Image Management and Domestic Network Optimization

### 11.1 Image Management Principles

The RustFS module will be deployed using Docker in the future.

Image management principles:

    Fix image versions.
    Do not use latest as a formal experimental version.
    Pull from official images first.
    Then tag to your Alibaba Cloud image repository or Harbor.
    Then push to private repositories.
    Subsequent notes will default to private repository image addresses.
    Retain image source documentation.
    Retain synchronization dates and versions.

---

### 11.2 Domestic Network Acceleration

Docker image pulls can use:

    mdocker acceleration address

Notes:

    Docker image acceleration improves stability for domestic network environments.
    Production environments are recommended to synchronize critical images to your Harbor or Alibaba Cloud image repository.
    Do not rely on real-time external pulls for all experiments.
    Do not use latest, as version changes can cause command, parameter, feature, and interface differences.

---

### 11.3 Image Sync Template

After confirming the RustFS image, use similar methods for synchronization:

    docker pull rustfs/rustfs:<fixed version>

    docker tag rustfs/rustfs:<fixed version> \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:<fixed version>

    docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:<fixed version>

Notes:

    Specific image names, tags, and startup parameters will be determined by subsequent validation.
    Do not hardcode versions based on memory.
    Do not use unverified versions directly as production notes.
    Production version selection must consider official Release Notes, known issues, licenses, feature completeness, community activity, and actual stress testing.

---

## 12. Security Design Principles

The RustFS module must reflect the following security principles:

    External access must use HTTPS.
    Internal HTTP is only allowed in trusted networks.
    Do not commit AccessKey / SecretKey to Git.
    Do not use root / admin accounts as long-term business credentials.
    Each business should use independent accounts or keys.
    Bucket permissions should follow the principle of minimal permissions.
    Grant deletion permissions cautiously.
    Reverse proxies should limit upload size, timeouts, and sources.
    Management entry points should not be exposed to the public internet.
    Monitor certificate expiration.
    Avoid logging sensitive keys in logs.
    Keys should be disabled and rotated after leaks.

---

## 13. Operations Design Principles

RustFS operations focus:

    Service status
    Container status
    Port listening
    API health checks
    Bucket count
    Object count
    Data directory capacity
    Node disk capacity
    Nginx access logs
    Nginx error logs
    Client 4xx / 5xx
    Upload failures
    Download failures
    Permission errors
    SignatureDoesNotMatch
    AccessDenied
    Certificate expiration
    Backup migration tasks

Common command directions:

    docker ps
    docker logs
    docker inspect
    ss -lntp
    curl -I
    mc alias list
    mc ls
    mc stat
    mc cp
    mc mirror
    df -hT
    du -sh
    journalctl
    nginx -t
    tail -f /var/log/nginx/access.log
    tail -f /var/log/nginx/error.log

---

## 14. Production Boundary Warnings

RustFS is a new object storage solution with high learning value, but production deployment must be cautious.

Production evaluation must cover:

    Current version maturity
    S3 API compatibility
    Multipart Upload compatibility
    SDK compatibility
    Large object upload stability
    Small object performance
    Node failure recovery
    Data recovery capability
    Permission system completeness
    HTTPS and reverse proxy compatibility
    Logging and monitoring capability
    Community activity
    Documentation completeness
    Version upgrade strategy
    License and enterprise compliance
    Backup migration capability
    Migration cost with existing MinIO / Ceph RGW

Do not recommend:

    Directly replacing production MinIO.
    Directly hosting core business object data.
    Launching without stress testing.
    Launching without recovery drills.
    Launching without monitoring alerts.
    Launching without backup migration plans.
    Launching without fixed versions.
    Launching without permission governance.

Recommendations:

    Start as a learning and validation environment.
    Then as a non-core test object storage.
    Then perform S3 compatibility testing.
    Then conduct failure drills.
    Then perform performance stress testing.
    Finally, evaluate suitability for production pilot.

---

## 15. RustFS and Advanced SRE Capabilities

The value of learning RustFS is not just starting an object storage container.

More importantly, it builds the following capabilities:

    New infrastructure component evaluation capability
    S3 compatibility verification capability
    Object storage production boundary judgment capability
    Docker single-node and multi-node deployment capability
    Reverse proxy and HTTPS entry design capability
    AccessKey / SecretKey permission governance capability
    Bucket / Object operations capability
    mc / AWS CLI toolchain capability
    Logging, capacity, port, and failure troubleshooting capability
    New-old object storage migration evaluation capability
    MinIO / Ceph RGW comparison and selection capability

From an advanced SRE perspective, the RustFS module's focus is not "replacing MinIO," but:

    How to validate, stress test, assess risks, and design production boundaries when facing new technologies.

---

## 16. Subsequent Note Output Requirements

RustFS subsequent notes must meet: /think

Module independence, readable separately.  
Each article has a clear experimental objective.  
Each article contains practical commands.  
Each article can be deployed in Docker / VM environment.  
No dependency on Kubernetes.  
No interference with MinIO, Ceph, or Longhorn experiments.  
Use 10.0.0.0/24 for independent planning.  
Use fixed version images.  
Record image sources.  
Demonstrate internal HTTP and external HTTPS.  
Demonstrate Nginx unified entry point.  
Demonstrate port planning.  
Demonstrate data directory planning.  
Demonstrate permissions and security.  
Demonstrate logs, capacity, and troubleshooting.  
Demonstrate production boundaries.  
Demonstrate official documentation references.

---

## 17. Stage Advancement Recommendations

Recommend proceeding in the following order:

    Step 1: 01 Clarify RustFS fundamentals, S3 object storage, and applicable scenarios.  
    Step 2: 02 Clarify differences between single-node and cluster modes.  
    Step 3: 03 Complete single-node Docker deployment and access verification.  
    Step 4: 04 Complete multi-node, multi-disk, and unified entry point practice.  
    Step 5: 05 System comparison with MinIO.  
    Step 6: 06 Client access using mc / AWS CLI.  
    Step 7: 07 Complete HTTPS, reverse proxy, and permission security design.  
    Step 8: 08 Complete logs, capacity, node anomalies, and recovery troubleshooting.  
    Step 9: 99 Complete stage summary and production boundary judgment.

---

## 18. Interview Expression Direction

If asked in an interview:

    What is RustFS? How do you view it?

You can respond:

    RustFS is an S3-compatible object storage system written in Rust, with a positioning similar to MinIO, mainly targeting object storage scenarios such as Bucket, Object, and S3 API. It is suitable for storing non-structured data like images, attachments, backup packages, log archives, AI datasets, and model files.  
    I would learn RustFS after MinIO, as MinIO's object storage model, mc tool, Bucket, AccessKey, reverse proxy, HTTPS, and backup migration methodology can be migrated to RustFS learning.  
    However, I would not directly treat RustFS as a mature production alternative. As a new object storage solution, it needs to focus on verifying version stability, S3 API compatibility, Multipart Upload, SDK compatibility, permission system, log monitoring, node failure recovery, backup migration, and production cases.  
    In experiments, I would first deploy single-node Docker to verify service startup, data directory, ports, Bucket creation, and object upload/download; then perform multi-node Docker / VM cluster deployment to verify nodes, disks, unified entry point, and failure recovery; finally provide external HTTPS entry via Nginx, with internal node communication using HTTP in a trusted network.  
    In operations, I would focus on logs, ports, capacity, AccessDenied, SignatureDoesNotMatch, Nginx 502/413/504, certificates, key management, and Bucket permissions. Before production use, pressure testing, recovery drills, and backup migration verification are mandatory.  
    Therefore, my attitude toward RustFS is: it can be used as a new S3 object storage solution for learning and verification, but before replacing MinIO or Ceph RGW in production, thorough testing and risk assessment are required.

---

## 19. Reference Documents

RustFS official website:

    https://rustfs.com/

RustFS official documentation:

    https://docs.rustfs.com/

RustFS GitHub:

    https://github.com/rustfs/rustfs

RustFS Docker image:

    https://hub.docker.com/r/rustfs/rustfs

AWS S3 API documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

MinIO official documentation:

    https://min.io/docs/minio/linux/index.html

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

Nginx official documentation:

    https://nginx.org/en/docs/

Docker official documentation:

    https://docs.docker.com/