# MinIO Directory Index

Recommended path: 05-Storage/02-MinIO/00-MinIO Directory Index.md

Tags: #MinIO #Object Storage #S3 #ErasureCoding #Distributed Storage #Docker #Nginx #HTTPS #Advanced SRE #Production Ops

---

## I. Module Overview

This document serves as a directory index for the MinIO module, providing a comprehensive learning path that covers everything from basic theory to practical deployment, distributed clusters, unified access points, permission management, data protection, monitoring and operations, and final summaries.

MinIO is an object storage system compatible with the S3 API, suitable for:

    Image storage
    Attachment storage
    Backup and archiving
    Log archiving
    Private object storage
    Storage of DevOps toolchain artifacts
    Object data storage in AI/data platforms

This module is designed as:

    A standalone learning resource for MinIO, independent of Ceph, Longhorn, and RustFS.
    Each note can be read and experimented with independently.
    Experiments are primarily conducted using Docker single-machine setups, Docker multi-node clusters, or VM multi-node clusters.
    The focus is on aligning with advanced SRE practices regarding object storage architecture understanding, deployment, fault recovery, permission governance, access point design, monitoring and alerts, and production-level operations.

---

## II. Module Learning Objectives

Upon completing this MinIO module, you should be able to:

1. Understand the basic concepts of object storage, the S3 protocol, Bucket, Object, AccessKey, and SecretKey.
2. Distinguish MinIO from traditional file systems, block storage, Ceph RGW, and cloud vendors' OSS/S3 solutions.
3. Comprehend MinIO's Erasure Coding data protection mechanism.
4. Master how to deploy MinIO on a single Docker machine.
5. Learn how to experiment with MinIO on a single-node multi-disk setup.
6. Understand how to deploy MinIO in a multi-node multi-disk distributed cluster environment.
7. Recognize the production-level design choices of using HTTP for internal node communication and HTTPS for external client access in MinIO.
8. Be able to set up Nginx to provide a unified HTTPS access point for MinIO.
9. Properly configure the 9000 API port, 9001 Console port, and reverse proxy settings.
10. Master the configuration of the mc client, Bucket management, object uploading/downloading, and policy management.
11. Understand how to manage permissions for MinIO users, Policies, AccessKeys, and SecretKeys.
12. Comprehend the recovery mechanisms for MinIO node failures, disk failures, and Erasure Coding issues.
13. Learn how to monitor key metrics such as Prometheus indicators, logs, capacity, and Bucket/Object growth.
14. Understand the concepts of mc mirroring, cross-cluster synchronization, and data migration.
15. Be able to assess whether a MinIO cluster meets basic production readiness criteria.

---

## III. Experimental Environment Planning

### 3.1 Experimental IP Range

The experimental environment for this module will use:

    10.0.0.0/24

Be sure to avoid using existing addresses:

| IP | Current Use |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |

---

### 3.2 MinIO Experimental Node Planning

It is recommended to use the following nodes for experiments:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO single-machine / distributed node 1 |
| 10.0.0.42 | minio-node02 | MinIO distributed node 2 |
| 10.0.0.43 | minio-node03 | MinIO distributed node 3 |
| 10.0.0.44 | minio-node04 | MinIO distributed node 4 |
| 10.0.0.45 | minio-client | mc client / stress testing / backup synchronization |
| 10.0.0.46 | minio-entry | Nginx unified access point / HTTPS reverse proxy (optional) |

Notes:

    For single-machine experiments, only 10.0.0.41 is required.
    For single-node multi-disk experiments, 10.0.0.41 can also be used.
    For 4-node multi-disk distributed experiments, use nodes 10.0.0.41-10.0.0.44The external entrance uses port 443.
It is not recommended to directly expose the HTTP service on port 9000 to the public network.### Nginx Entry Point
### Data Synchronization and Migration
### Fault Recovery for Distributed Clusters