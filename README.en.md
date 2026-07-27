# Ops Skill Tree

![DevOps](https://img.shields.io/badge/DevOps-Knowledge-blue)  
![Linux](https://img.shields.io/badge/Linux-Expert-green)  
![Kubernetes](https://img.shields.io/badge/Kubernetes-CloudNative-blue)  
![License](https://img.shields.io/badge/license-MIT-green)

A personal knowledge library project for operations engineers.

This repository is designed to build a comprehensive **Ops Skill Tree**, organizing knowledge in areas such as Linux, Docker, Kubernetes, networking, storage, middleware, CI/CD, monitoring, logging, security, GPU, and virtualization.

All content is maintained using **Markdown + Obsidian**.

---

# Project Goals

To create a complete **operations technology knowledge library**:

- Document daily operations practices

- Accumulate technical experience

- Build a knowledge graph

- Construct an Ops Skill Tree

- Organize Runbook operation manuals

- Collect production failure cases

Ultimately, the goal is to form a **systematized operations knowledge framework**.

---

# Ops Skill Tree

![Ops Skill Tree](attachments/images/ops-skill-tree.png)

```text
Ops Skill Tree
│
├ Linux
├ Docker
├ Kubernetes
├ Networking
├ Storage
├ Middleware
├ CI/CD
├ Monitoring
│
├ Logging
│
├ Security
│
├ GPU
│
├ Virtualization
│
├ Cloud
└ SRE
```

---

# Cloud Native Architecture

![Cloud Native Architecture](attachments/images/cloud-native-architecture.png)

```text
Developer
    │
    ▼
Git Repository
    │
    ▼
CI Pipeline
(Jenkins / GitLab CI)
    │
    ▼
Image Build
(Docker)
    │
    ▼
Image Registry
(Harbor)
    │
    ▼
Kubernetes Cluster
    │
    ▼
Ingress / Gateway
    │
    ▼
Service
    │
    ▼
Pod
```

---

# Kubernetes Data Flow

![K8s Network Flow](attachments/images/k8s-network-flow.png)

```text
User
 │
 ▼
Ingress
 │
 ▼
Service
 │
 ▼
Pod
 │
 ▼
Container
```

---

# Project Structure

```text
ops-skill-tree
│
├ 00-Operations Architecture
│
├ 01-Windows
│
├ 02-Linux
│
├ 03-Docker
│
├ 04-Kubernetes
│
├ 05-Storage
│
├ 06-Networking
│
├ 07-Middleware
│
├ 08-CI/CD
│
├ 09-Monitoring
│
├ 10-Logging
│
├ 11-GPU
│
├ 12-Virtualization
│
├ 13-Cloud Platforms
│
├ 14-Security
│
├ 15-Container Networking
│
├ 16-Operations Tools
│
├ 17-PVE
│
├ 18-SRE
│
├ 19-Failure Cases
│
├ 20-Runbooks
│
├ scripts
│
└ attachments
```

---

# Module Descriptions

## 00 Operations Architecture

Overall technology framework:

- Operations knowledge system

- Cloud native architecture

- Operations learning path

---

## 01 Windows

Windows operations:

- Windows architecture

- PowerShell

- Windows Terminal

- WSL

- Troubleshooting

---

## 02 Linux

Linux systems:

- File systems

- User permissions

- Process management

- systemd

- SELinux

- Performance analysis

- Logging systems

---

## 03 Docker

Container technology:

- Docker architecture

- Images

- Containers

- Networking

- Storage

- Dockerfile

- Docker Compose

---

## 04 Kubernetes

Core cloud native components:

- Pods

- Deployments

- StatefulSets

- DaemonSets

- Services

- Ingresses

- Helm

- Storage

---

## 05 Storage

Storage systems:

- NFS

- Ceph

- GlusterFS

- MinIO

- PVs

- PVCs

- StorageClasses

---

## 06 Networking

Network fundamentals:

- TCP/IP

- VLANs

- VXLANs

- BGP

- DNS

- Network troubleshooting

---

## 07 Middleware

Service components:

- Nginx

- MySQL

- Redis

- Kafka

- RabbitMQ

- Elasticsearch

- Nacos

- Consul

---

## 08 CI/CD

Continuous integration:

- Jenkins

- GitLab CI

- ArgoCD

- Tek