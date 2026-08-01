# 98-Experiment Environment Description.md

## Document Overview
This document records the experimental environment baseline for the 08-CI-CD series course, as well as the standard operational steps to configure containerd 2.2.1 for Harbor (HTTP:8090) on Kubernetes nodes.  
This document only retains the operation commands that have been verified and validated in the current environment, making it suitable for subsequent experiments and production reference.

## Tags
#CICD #GitLab #Jenkins #Harbor #Kubernetes #containerd #Helm #ExperimentalEnvironment

---

## I. Experimental Environment Baseline

### 1. CI/CD Platform Node
- ops-server:10.0.0.10
- Deployed Components:
  - GitLab
  - Jenkins
  - Harbor

### 2. Kubernetes Cluster Nodes
- k8s-master:10.0.0.20
- k8s-node01:10.0.0.21
- k8s-node02:10.0.0.22

### 3. Version Information
- OS:Ubuntu 22.04.5 LTS
- Kubernetes:v1.31.14
- Container Runtime:containerd 2.2.1

### 4. Harbor Access Address
- Web:http://10.0.0.10:8090
- Image Repository Prefix:10.0.0.10:8090

---

## II. Problem Scenario

Currently, Harbor provides service via HTTP + 8090 port.  
By default, Docker and containerd will prioritize HTTPS repository access for private registries, resulting in errors similar to the following:

    http: server gave HTTP response to HTTPS client

The goal of this environment is:

1. Allow Docker on ops-server to push images to Harbor
2. Allow containerd on Kubernetes nodes to pull images from Harbor
3. Provide a unified image distribution foundation for subsequent Jenkins, GitLab CI, Helm, Argo CD

---

## III. Trust Harbor (ops-server) with Docker

This section's operations are executed on `10.0.0.10`.

### 1. Backup Docker Configuration
If `/etc/docker/daemon.json` exists, back it up first:

    sudo test -f /etc/docker/daemon.json && \
    sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak.$(date +%F-%H%M%S)

### 2. Configure insecure registry
If currently only need to trust Harbor, directly write:

    sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
    {
      "insecure-registries": ["10.0.0.10:8090"]
    }
    EOF

If the file already has other configurations, modify while preserving the original valid JSON structure and add:

    "insecure-registries": ["10.0.0.10:8090"]

### 3. Restart Docker

    sudo systemctl restart docker
    sudo systemctl status docker --no-pager

### 4. Verify Docker Configuration

    docker info | grep -A5 "Insecure Registries"

---

## IV. Trust Harbor (Kubernetes Nodes) with containerd

This section's operations need to be executed on each Kubernetes node:

- 10.0.0.20
- 10.0.0.21
- 10.0.0.22

containerd officially recommends pointing `config_path` to the directory where `hosts.toml` resides in 2.x versions.:contentReference[oaicite:1]{index=1}

### 1. Backup containerd Configuration

    sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%F-%H%M%S)

### 2. Converge registry config_path to a single path

Execute:

    sudo sed -i "s#config_path = '/etc/containerd/certs.d:/etc/docker/certs.d'#config_path = '/etc/containerd/certs.d'#g" /etc/containerd/config.toml

Verify:

    grep -n "config_path" /etc/containerd/config.toml

Expected result includes:

    config_path = '/etc/containerd/certs.d'

### 3. Create Harbor registry directory

containerd's official hosts directory mode supports organizing directories by registry hostname; for registry with port on Unix, different naming forms may exist, so this environment uniformly creates the following two directories to ensure hit. :contentReference[oaicite:2]{index=2}

    sudo mkdir -p /etc/containerd/certs.d/10.0.0.10_8090_
    sudo mkdir -p /etc/containerd/certs.d/10.0.0.10:8090

### 4. Write hosts.toml

    sudo tee /etc/containerd/certs.d/10.0.0.10_8090_/hosts.toml > /dev/null <<'EOF'
    server = "http://10.0.0.10:8090"

    [host."http://10.0.0.10:8090"]
      capabilities = ["pull", "resolve", "push"]
      skip_verify = true
    EOF

### 5. Synchronize to colon directory

    sudo cp /etc/containerd/certs.d/10.0.0.10_8090_/hosts.toml /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml

### 6. Check hosts.toml

```
cat /etc/containerd/certs.d/10.0.0.10_8090_/hosts.toml
cat /etc/containerd/certs.d/10.0.0.10:8090/hosts.toml

### 7. Restart containerd

    sudo systemctl restart containerd
    sudo systemctl status containerd --no-pager

---

## Five. Push Test Image to Harbor

This section is executed on `10.0.0.10`.

### 1. Log in to Harbor

    docker login 10.0.0.10:8090

### 2. Tag Local Image

Example:

    docker tag busybox:1.35 10.0.0.10:8090/demo/busybox:1.35

### 3. Push Image

    docker push 10.0.0.10:8090/demo/busybox:1.35

---

## Six. Verify Kubernetes Node Pulling Harbor Image

Note:

- This section prioritizes verification using `crictl`
- `crictl` is a CRI-compatible runtime debugging tool, closer to the actual image pulling chain of kubelet
- `ctr` is not used as the final judgment basis

Kubernetes official documentation states that `crictl` is used for debugging CRI runtime on nodes.:contentReference[oaicite:3]{index=3}

### 1. Pull Image on Node

    sudo crictl pull 10.0.0.10:8090/demo/busybox:1.35

### 2. Success Determination
If output is similar to:

    Image is up to date

It indicates that containerd on this node can normally pull images from Harbor.

---

## Seven. Verify Kubernetes Pod Pulling Harbor Image

After `crictl pull` is successful at the node level, verify at the Pod level.

### 1. Create Test Pod

    kubectl run harbor-test \
      --image=10.0.0.10:8090/demo/busybox:1.35 \
      --image-pull-policy=Always \
      --command -- sleep 3600

### 2. Check Pod Status

    kubectl get pod harbor-test -o wide
    kubectl describe pod harbor-test

### 3. Delete Test Pod

    kubectl delete pod harbor-test

---

## Eight. Current Configuration Impact Scope

### 1. Current Configuration Only Affects Harbor
This configuration only adds trust capability for the following private repositories:

    10.0.0.10:8090

### 2. Does Not Change Default Public Image Sources
If the image is written as:

    10.0.0.10:8090/demo/busybox:1.35

It will pull from Harbor.

If the image is written as:

    busybox:1.35
    nginx:1.27

In the form without explicit repository address, the default parsing logic remains unchanged, generally pointing to public repositories (usually docker.io).

### 3. When Will Default Pull from Harbor
Only when the following configurations are added later will the default image source behavior change:

- registry mirror
- default image source replacement
- proxy cache repository
- offline repository proxy strategy

These configurations are not performed in this experiment.

---

## Nine. Current Stage Conclusion

In this experimental environment, the following chain has been verified to be functional:

Local Image
-> Harbor
-> Kubernetes Node containerd
-> Kubernetes Pod

This is the common prerequisite for subsequent content:

- Jenkins Pipeline image push
- GitLab CI image push
- Helm deployment to Kubernetes
- Argo CD GitOps declarative delivery /think
```