# 98-Experimental Environment Description.md

## Document Description
This document records the experimental environment baseline for the 08-CI-CD series of courses, as well as the standard steps to configure containerd 2.2.1 repository trust for Harbor (HTTP:8090) on Kubernetes nodes.
This document only retains operation commands that have been verified in the current environment and can serve as a reference for subsequent experiments and production.

## Tags
#CICD #GitLab #Jenkins #Harbor #Kubernetes #containerd #Helm #Experimental Environment

---

## I. Experimental Environment Baseline

### 1. CI/CD Platform Nodes
- ops-server: 10.0.0.10
- Deployed Components:
  - GitLab
  - Jenkins
  - Harbor

### 2. Kubernetes Cluster Nodes
- k8s-master: 10.0.0.20
- k8s-node01: 10.0.0.21
- k8s-node02: 10.0.0.22

### 3. Version Information
- OS: Ubuntu 22.04.5 LTS
- Kubernetes: v1.31.14
- Container Runtime: containerd 2.2.1

### 4. Harbor Access Address
- Web: http://10.0.0.10:8090
- Image Repository Prefix: 10.0.0.10:8090

---

## II. Problem Scenario

Currently, Harbor provides services using the HTTP + 8090 port.
By default, both Docker and containerd will prefer to access private registries via HTTPS repositories, which can result in errors such as:

    http: server gave HTTP response to HTTPS client

The goal of this environment is to:

1. Allow Docker on ops-server to push images to Harbor
2. Enable containerd on Kubernetes nodes to pull images from Harbor
3. Provide a unified image distribution foundation for subsequent Jenkins, GitLab CI, Helm, and Argo CD processes

---

## III. Docker Trusting Harbor (ops-server)

These operations are performed on `10.0.0.10`.

### 1. Back up Docker Configuration
If `/etc/docker/daemon.json` already exists, back it up first:

    sudo test -f /etc/docker/daemon.json && \
    sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak.$(date +%F-%H%M%S)

### 2. Configure Insecure Registry
If you only need to trust Harbor at this time, you can directly write the following:

    sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
    {
      "insecure-registries": ["10.0.0.10:8090"]
    }
    EOF

If the file already contains other configurations, make sure to retain the original valid JSON structure and add the new entry:

    "insecure-registries": ["10.0.0.10:8090"]

### 3. Restart Docker

    sudo systemctl restart docker
    sudo systemctl status docker --no-pager

### 4. Verify Docker Configuration

    docker info | grep -A5 "Insecure Registries"

---

## IV. containerd Trusting Harbor (Kubernetes Nodes)

These operations need to be performed on each Kubernetes node:

- 10.0.0.20
- 10.0.0.21
- 10.0.0.22

Containerd officially recommends using `config_path` to point to the directory where `hosts.toml` is located in version 2.x.:contentReference[oaicite:1]{index=1}

### 1. Back up containerd Configuration

    sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%F-%H%M%S)

### 2. Set the registry config_path to a Single Path

Execute:

    sudo sed -i "s#config_path = '/etc/containerd/certs.d:/etc/docker/certs.d'#config_path = '/etc/containerd/certs.d'#g" /etc/containerd/config.toml

Verify:

    grep -n "config_path" /etc/containerd/config.toml

The expected result should include:

    config_path = '/etc/containerd/certs.d'

### 3. Create Harbor Registry Directories

Containerd's official hosts directory structure supports organizing directories by registry hostname; however, registries with ports may use different naming conventions on Unix systems. Therefore, in this environment, the following two directories are created to ensure proper matching.:contentReference[oaicite:2]{index=2}

    sudo mkdir -If the image is listed in the following format:

    busybox:1.35
    nginx:1.27

Without an explicit repository address, it will still be parsed according to the default logic, typically pointing to public repositories (usually docker.io).

### 3. Under What Circumstances Will Harbor Be Used by Default?
The default behavior for image sourcing will only change if the following additional configurations are set:

- registry mirror
- default image source replacement
- proxy cache repository
- offline repository proxy strategy

None of these configurations have been applied in this experiment.

---

## IX. Conclusions at This Stage

In this experimental environment, it has been verified that the following pathways can be established:

Local image
-> Harbor
-> Kubernetes node containerd
-> Kubernetes Pod

This lays the foundation for the subsequent implementation of the following tasks:

- Jenkins Pipeline for image deployment
- GitLab CI for image delivery
- Helm for publishing to Kubernetes
- Argo CD for declarative delivery based on GitOps