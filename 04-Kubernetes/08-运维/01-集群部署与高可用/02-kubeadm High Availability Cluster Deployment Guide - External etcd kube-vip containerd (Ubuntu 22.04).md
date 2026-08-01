# 02-kubeadm High Availability Cluster Deployment Practice: External etcd + kube-vip + containerd (Ubuntu 22.04)

Recommended path:

    04-Kubernetes/08-Operations/01-Cluster Deployment and High Availability/02-kubeadm High Availability Cluster Deployment Practice: External etcd + kube-vip + containerd (Ubuntu 22.04).md

Tags:

    #Kubernetes
    #kubeadm
    #HighAvailableClusters
    #External-etcd
    #kube-vip
    #containerd
    #IPVS
    #Ubuntu2204
    #DomesticEnvironment
    #ClusterDeployment

---

## I. Document Objectives

This document records the deployment process of a Kubernetes high availability cluster based on kubeadm.

This solution is tailored for domestic environments, adopting an External etcd architecture:

    Ubuntu Server 22.04
    containerd
    kubeadm / kubelet / kubectl
    Alibaba Cloud Docker CE source
    Alibaba Cloud Kubernetes source
    Alibaba Cloud google_containers image repository
    3 Master high availability control plane
    3 External etcd independent nodes
    kube-vip provides APIServer VIP
    kube-proxy IPVS mode
    containerd data directory adjusted to /data/containerd
    etcd data directory adjusted to /data/etcd

The objective of this document is to create a Kubernetes External etcd high availability deployment notes that can be executed subsequently.

---

## II. Node Planning

### 2.1 Cluster Planning

| Item | Planning |
|---|---|
| Operating System | Ubuntu Server 22.04 |
| Kubernetes Version | v1.31.14 |
| Container Runtime | containerd |
| Deployment Tool | kubeadm |
| Control Plane | 3 Master |
| etcd Mode | External etcd |
| etcd Nodes | 3 independent etcd nodes |
| APIServer High Availability | kube-vip |
| APIServer VIP | 10.0.0.30 |
| APIServer Domain | k8s-api-server |
| Pod Network Segment | 10.244.0.0/16 |
| Service Network Segment | 10.96.0.0/12 |
| kube-proxy Mode | IPVS |
| CNI Plugin | Calico |
| containerd Data Directory | /data/containerd |
| etcd Data Directory | /data/etcd |

---

### 2.2 Node IP Planning

| Role | Hostname | IP Address | Notes |
|---|---|---:|---|
| Operations Tool Node | ops-server | 10.0.0.10 | GitLab / Jenkins / Harbor, optional |
| Master 1 | k8s-master-01 | 10.0.0.20 | control-plane |
| Master 2 | k8s-master-02 | 10.0.0.21 | control-plane |
| Master 3 | k8s-master-03 | 10.0.0.22 | control-plane |
| Worker 1 | k8s-worker-01 | 10.0.0.23 | Worker node |
| Worker 2 | k8s-worker-02 | 10.0.0.24 | Worker node |
| Worker 3 | k8s-worker-03 | 10.0.0.25 | Worker node, optional |
| APIServer VIP | k8s-api-server | 10.0.0.30 | kube-vip floating IP |
| etcd 1 | etcd-01 | 10.0.0.31 | External etcd |
| etcd 2 | etcd-02 | 10.0.0.32 | External etcd |
| etcd 3 | etcd-03 | 10.0.0.33 | External etcd |

Notes:

    10.0.0.30 is not assigned to any real server.
    10.0.0.30 is only used as the virtual IP for kube-vip.
    kubeadm init and kubeadm join access the control plane through k8s-api-server:6443.
    etcd-01, etcd-02, etcd-03 are not added as Kubernetes Worker nodes to the cluster.
    External etcd is managed by the local kubelet on etcd nodes as static Pods.

---

### 2.3 Architecture Diagram /think

kubectl / kubelet / Operations Tools
                  |
                  |
          k8s-api-server:6443
                  |
                  |
             VIP:10.0.0.30
                  |
                  |
              kube-vip
                  |
    +-------------+-------------+-------------+
    |                           |             |
    |                           |             |
    v                           v             v
    k8s-master-01               k8s-master-02  k8s-master-03
    10.0.0.20                   10.0.0.21      10.0.0.22
    control-plane               control-plane  control-plane
    kube-apiserver              kube-apiserver kube-apiserver
    kube-scheduler              kube-scheduler kube-scheduler
    kube-controller-manager     kube-controller-manager
    kube-vip                    kube-vip       kube-vip

    +-------------+-------------+-------------+
    |                           |             |
    |                           |             |
    v                           v             v
    etcd-01                     etcd-02        etcd-03
    10.0.0.31                   10.0.0.32      10.0.0.33
    etcd                        etcd           etcd
    static Pod                  static Pod     static Pod
    /data/etcd                  /data/etcd     /data/etcd

    +-------------+-------------+-------------+
    |                           |             |
    |                           |             |
    v                           v             v
    k8s-worker-01               k8s-worker-02  k8s-worker-03
    10.0.0.23                   10.0.0.24      10.0.0.25
    worker                      worker         worker
    kubelet                     kubelet        kubelet
    kube-proxy                  kube-proxy     kube-proxy
    containerd                  containerd     containerd

---

## Three. Pre-deployment Checks

All nodes confirmation:

    ip addr
    hostname
    free -h
    df -h
    timedatectl
    cat /etc/os-release

Confirmation items:

    1. All nodes have fixed IP addresses
    2. All nodes have unique hostnames
    3. All nodes have synchronized time
    4. All nodes can communicate with each other
    5. All nodes can access domestic software repositories
    6. 10.0.0.30 is not occupied by any host
    7. /data is recommended to be mounted to an independent data disk or large capacity partition
    8. Master nodes can access etcd nodes on port 2379
    9. etcd nodes can communicate with each other on ports 2379 and 2380

Check node communication:

    ping -c 3 10.0.0.20
    ping -c 3 10.0.0.21
    ping -c 3 10.0.0.22
    ping -c 3 10.0.0.23
    ping -c 3 10.0.0.24
    ping -c 3 10.0.0.25
    ping -c 3 10.0.0.31
    ping -c 3 10.0.0.32
    ping -c 3 10.0.0.33

Check /data:

    sudo mkdir -p /data
    df -h /data

Notes:

    If /data is not mounted independently, but is just a regular directory under the root partition, then containerd and etcd data will still occupy system disk space.
    In production environments, it is recommended to mount /data to an independent data disk.

---

## Four. Configure Hostnames and hosts on All Nodes

The following operations are performed on all Master, Worker, and etcd nodes.

Node range:

    k8s-master-01
    k8s-master-02
    k8s-master-03
    k8s-worker-01
    k8s-worker-02
    k8s-worker-03
    etcd-01
    etcd-02
    etcd-03

---

### 4.1 Set Hostname

Execute on k8s-master-01:

    sudo hostnamectl set-hostname k8s-master-01

Execute on k8s-master-02:

    sudo hostnamectl set-hostname k8s-master-02

Execute on k8s-master-03:

sudo hostnamectl set-hostname k8s-master-03

Run on k8s-worker-01:

    sudo hostnamectl set-hostname k8s-worker-01

Run on k8s-worker-02:

    sudo hostnamectl set-hostname k8s-worker-02

Run on k8s-worker-03:

    sudo hostnamectl set-hostname k8s-worker-03

Run on etcd-01:

    sudo hostnamectl set-hostname etcd-01

Run on etcd-02:

    sudo hostnamectl set-hostname etcd-02

Run on etcd-03:

    sudo hostnamectl set-hostname etcd-03

Check:

    hostname

---

### 4.2 Configure /etc/hosts

Run on all nodes:

    sudo cp /etc/hosts /etc/hosts.bak.$(date +%F-%H%M%S)

    cat <<EOF | sudo tee -a /etc/hosts

    10.0.0.10 ops-server

    10.0.0.20 k8s-master-01
    10.0.0.21 k8s-master-02
    10.0.0.22 k8s-master-03

    10.0.0.23 k8s-worker-01
    10.0.0.24 k8s-worker-02
    10.0.0.25 k8s-worker-03

    10.0.0.30 k8s-api-server

    10.0.0.31 etcd-01
    10.0.0.32 etcd-02
    10.0.0.33 etcd-03
    EOF

Check resolution:

    getent hosts k8s-api-server
    getent hosts k8s-master-01
    getent hosts k8s-master-02
    getent hosts k8s-master-03
    getent hosts etcd-01
    getent hosts etcd-02
    getent hosts etcd-03

---

## Five. System Initialization on All Nodes

The following operations are executed on all Master, Worker, and etcd nodes.

---

### 5.1 Configure Clock Synchronization

Install chrony:

    sudo apt update
    sudo apt install -y chrony

Replace with domestic NTP source:

    sudo sed -i.bak '/^pool ntp\.ubuntu\.com\|^pool [012]\.ubuntu\.pool\.ntp\.org/s/^/#/;/^#pool ntp\.ubuntu\.com/i\pool ntp.aliyun.com iburst\npool ntp.huaweicloud.com iburst\npool ntp.tencent.com iburst\npool time.windows.com iburst' /etc/chrony/chrony.conf

Start and set to boot:

    sudo systemctl enable --now chrony

Check:

    timedatectl
    chronyc sources -v

---

### 5.2 Disable Swap

Temporarily disable Swap:

    sudo swapoff -a

Permanently disable Swap:

    sudo sed -i.bak '/swap/s/^/#/' /etc/fstab

Check:

    free -h
    swapon --show

If swapon --show shows no output, Swap is disabled.

---

### 5.3 Load Kernel Modules

Write kernel module configuration:

    cat <<EOF | sudo tee /etc/modules-load.d/ipvs.conf
    overlay
    br_netfilter
    ip_vs
    ip_vs_rr
    ip_vs_wrr
    ip_vs_sh
    nf_conntrack
    EOF

Load modules immediately:

    sudo modprobe overlay
    sudo modprobe br_netfilter
    sudo modprobe ip_vs
    sudo modprobe ip_vs_rr
    sudo modprobe ip_vs_wrr
    sudo modprobe ip_vs_sh
    sudo modprobe nf_conntrack

Check:

    lsmod | grep -E "overlay|br_netfilter|ip_vs|nf_conntrack"

Notes:

    overlay and br_netfilter are foundational requirements for containerd and Kubernetes networking.
    ip_vs, ip_vs_rr, ip_vs_wrr, ip_vs_sh, and nf_conntrack are used for kube-proxy IPVS mode.
    This document will explicitly configure kube-proxy to use IPVS mode later.

---

### 5.4 Configure Kernel Parameters

Write sysctl configuration:

    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

Load configuration:

    sudo sysctl --system

Check:

    sysctl net.bridge.bridge-nf-call-iptables
    sysctl net.bridge.bridge-nf-call-ip6tables
    sysctl net.ipv4.ip_forward

Expected results:

    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward = 1

---

### 5.5 Install Basic Dependencies

Install Common Dependencies for Kubernetes Nodes:

    sudo apt-get install -y \
      ipset \
      ipvsadm \
      nfs-common \
      conntrack \
      socat \
      apt-transport-https \
      ca-certificates \
      curl \
      gnupg \
      wget \
      vim \
      net-tools \
      lsof

Notes:

    conntrack is a connection tracking tool, and kube-proxy depends on it.
    socat is commonly used for kubectl exec, kubectl port-forward, and similar functions.
    ipset and ipvsadm are used for troubleshooting kube-proxy IPVS mode.
    nfs-common is used for subsequent NFS-type storage integration.

---

## Six. Installing containerd on All Nodes

The following operations are executed on all Master, Worker, and etcd nodes.

This article uses Alibaba Cloud's Docker CE source to install containerd.io.

Notes:

    Kubernetes nodes use containerd as the container runtime.
    This article only installs containerd.io and does not install the full Docker Engine.
    The containerd data directory is uniformly adjusted to /data/containerd.

---

### 6.1 Configuring Alibaba Cloud Docker CE Source

Create the keyrings directory:

    sudo install -m 0755 -d /etc/apt/keyrings

Add Docker GPG key:

    curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | \
      sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

Set permissions:

    sudo chmod a+r /etc/apt/keyrings/docker.gpg

Write Docker CE software source:

    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

Update apt cache:

    sudo apt-get update

---

### 6.2 Install containerd.io

Install:

    sudo apt-get install -y containerd.io

Check version:

    containerd --version

---

### 6.3 Generate containerd Default Configuration

Create configuration directory:

    sudo mkdir -p /etc/containerd

Generate default configuration:

    containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

Backup configuration:

    sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%F-%H%M%S)

---

### 6.4 Modify containerd Image Storage Location

Create directory:

    sudo mkdir -p /data/containerd

Modify containerd root path:

    sudo sed -i.bak 's#^root = "/var/lib/containerd"#root = "/data/containerd"#g' /etc/containerd/config.toml

Check:

    grep -n '^root = ' /etc/containerd/config.toml

Expected result:

    root = "/data/containerd"

Notes:

    root is the persistent data directory for containerd.
    Image, snapshots, metadata, and other main data are stored in this directory.
    state defaults to /run/containerd, which is a runtime state directory and generally does not need modification.
    It is best to complete this configuration before kubeadm init.

---

### 6.5 Enable SystemdCgroup

Modify configuration:

    sudo sed -i.bak 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

Check:

    grep "SystemdCgroup" /etc/containerd/config.toml

Expected result:

    SystemdCgroup = true

---

### 6.6 Replace pause Image with Alibaba Cloud Source

Replace the pause image in containerd configuration with the Alibaba Cloud image:

    sudo sed -i.bak -E 's#registry.k8s.io/pause:[0-9.]+#registry.aliyuncs.com/google_containers/pause:3.10#g' /etc/containerd/config.toml

Verify Sandbox image configuration:

    containerd config dump | grep -E "sandbox_image|pinned_helper_image"

If registry.k8s.io/pause is still visible, manually check:

    grep -n "pause" /etc/containerd/config.toml

---

### 6.7 Restart containerd

Restart:

    sudo systemctl restart containerd

Set to start on boot:

    sudo systemctl enable containerd

Check status:

    systemctl status containerd --no-pager

Check data directory:

    sudo ls -ld /data/containerd

---

### 6.8 Configure crictl

Write crictl configuration: /think

```bash
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

Check:

    sudo crictl info

View images:

    sudo crictl images

---

## Seven. Install nerdctl on all nodes

nerdctl is a Docker-style CLI tool for containerd, suitable for troubleshooting containerd images, containers, and namespaces.

The following operations are performed on all Master, Worker, and etcd nodes.

---

### 7.1 Install nerdctl minimal package

Set version:

    NERDCTL_VERSION=2.2.2

Download:

    cd /tmp
    wget https://github.com/containerd/nerdctl/releases/download/v${NERDCTL_VERSION}/nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz

Install:

    sudo tar -C /usr/local/bin -xzf nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz

Check:

    nerdctl --version

If GitHub download is slow, you can download the nerdctl tar package in advance and manually upload it to the nodes for extraction.

---

### 7.2 Common check commands

View images in the default namespace:

    sudo nerdctl images

View images in the k8s.io namespace used by Kubernetes:

    sudo nerdctl -n k8s.io images

View Kubernetes containers:

    sudo nerdctl -n k8s.io ps

View all Kubernetes containers:

    sudo nerdctl -n k8s.io ps -a

---

## Eight. Install kubeadm, kubelet, kubectl on all nodes

The following operations are performed on all Master, Worker, and etcd nodes.

This document uses the Alibaba Cloud Kubernetes v1.31 software source.

Note:

    etcd nodes need kubelet and kubeadm to generate and run etcd static Pods.
    Installing kubectl on etcd nodes is not mandatory, but installing it uniformly facilitates operations and troubleshooting.

---

### 8.1 Configure Alibaba Cloud Kubernetes source

Install base dependencies:

    sudo apt-get update

    sudo apt-get install -y \
      apt-transport-https \
      ca-certificates \
      curl \
      gpg

Create keyrings directory:

    sudo mkdir -p /etc/apt/keyrings

Add Kubernetes GPG key:

    curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.31/deb/Release.key | \
      sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

Add Kubernetes v1.31 apt source:

    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.31/deb/ /" | \
      sudo tee /etc/apt/sources.list.d/kubernetes.list

Update apt cache:

    sudo apt-get update

---

### 8.2 View installable versions

View versions:

    apt-cache madison kubeadm
    apt-cache madison kubelet
    apt-cache madison kubectl

This document uses:

    v1.31.14

If 1.31.14-1.1 appears in apt-cache madison, it is recommended to specify installing this version.

---

### 8.3 Install kubeadm, kubelet, kubectl

Install specific versions:

    sudo apt-get install -y \
      kubelet=1.31.14-1.1 \
      kubeadm=1.31.14-1.1 \
      kubectl=1.31.14-1.1

If the source does not have 1.31.14-1.1, you can use the latest patch version from the current v1.31 source:

    sudo apt-get install -y kubelet kubeadm kubectl

Check versions:

    kubeadm version
    kubelet --version
    kubectl version --client

---

### 8.4 Lock versions

Lock Kubernetes component versions:

    sudo apt-mark hold kubelet kubeadm kubectl

Check:

    apt-mark showhold

---

### 8.5 Enable kubelet

Enable kubelet:

    sudo systemctl enable kubelet

Check status:

    systemctl status kubelet --no-pager

Note:

    On Master and Worker nodes, kubelet requires kubeadm init or kubeadm join to obtain complete configuration.
    On etcd nodes, kubelet will be separately configured as an etcd static Pod manager later.

---

## Nine. Deploy External etcd Cluster

The following operations are performed only on etcd-01, etcd-02, etcd-03.

External etcd essence:

    etcd nodes do not join the Kubernetes cluster
    etcd is managed by local kubelet as a static Pod
    Kubernetes Master accesses etcd via https://10.0.0.31:2379、https://10.0.0.32:2379、https://10.0.0.33:2379

---

### 9.1 Configure kubelet to manage static Pod on etcd nodes

Execute on etcd-01, etcd-02, etcd-03:

    sudo mkdir -p /etc/systemd/system/kubelet.service.d /think

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
[Service]
ExecStart=
ExecStart=/usr/bin/kubelet \\
  --address=127.0.0.1 \\
  --pod-manifest-path=/etc/kubernetes/manifests \\
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock
Restart=always
EOF
```

Create static Pod directory and etcd data directory:

```bash
sudo mkdir -p /etc/kubernetes/manifests
sudo mkdir -p /data/etcd
```

Restart kubelet:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl enable kubelet
```

Check:

```bash
systemctl status kubelet --no-pager
```

Note:

- The kubelet on etcd nodes is not used to join Kubernetes clusters.
- It only listens to /etc/kubernetes/manifests and launches local etcd static Pods.

---

### 9.2 Generate etcd CA on etcd-01

Execute on etcd-01:

```bash
sudo kubeadm init phase certs etcd-ca
```

Check:

```bash
sudo ls -l /etc/kubernetes/pki/etcd/
```

Should show:

```
ca.crt
ca.key
```

---

### 9.3 Distribute etcd CA to etcd-02 and etcd-03

Execute on etcd-01:

```bash
sudo tar -C /etc/kubernetes/pki/etcd -czf /root/etcd-ca.tar.gz ca.crt ca.key

scp /root/etcd-ca.tar.gz root@etcd-02:/root/
scp /root/etcd-ca.tar.gz root@etcd-03:/root/
```

Execute on etcd-02 and etcd-03:

```bash
sudo mkdir -p /etc/kubernetes/pki/etcd
sudo tar -C /etc/kubernetes/pki/etcd -xzf /root/etcd-ca.tar.gz
```

Check:

```bash
sudo ls -l /etc/kubernetes/pki/etcd/
```

Should show:

```
ca.crt
ca.key
```

Note:

- This temporarily distributes ca.key to generate each etcd node's server/peer/healthcheck-client certificates.
- In production environments, ca.key must be strictly protected.
- After certificate generation, it's recommended to retain the key only in controlled nodes or secure backup locations.

---

### 9.4 Generate kubeadm-etcd Configuration on Each etcd Node

Execute on etcd-01:

```bash
ETCD_NAME="etcd-01"
ETCD_IP="10.0.0.31"
```

Execute on etcd-02:

```bash
ETCD_NAME="etcd-02"
ETCD_IP="10.0.0.32"
```

Execute on etcd-03:

```bash
ETCD_NAME="etcd-03"
ETCD_IP="10.0.0.33"
```

Then execute the following on all three etcd nodes:

```bash
ETCD_INITIAL_CLUSTER="etcd-01=https://10.0.0.31:2380,etcd-02=https://10.0.0.32:2380,etcd-03=https://10.0.0.33:2380"

cat <<EOF | sudo tee /root/kubeadm-etcd.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  name: "${ETCD_NAME}"
localAPIEndpoint:
  advertiseAddress: "${ETCD_IP}"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "v1.31.14"
imageRepository: "registry.aliyuncs.com/google_containers"
etcd:
  local:
    dataDir: "/data/etcd"
    serverCertSANs:
    - "${ETCD_NAME}"
    - "${ETCD_IP}"
    peerCertSANs:
    - "${ETCD_NAME}"
    - "${ETCD_IP}"
    extraArgs:
    - name: name
      value: "${ETCD_NAME}"
    - name: listen-client-urls
      value: "https://${ETCD_IP}:2379,https://127.0.0.1:2379"
    - name: advertise-client-urls
      value: "https://${ETCD_IP}:2379"
    - name: listen-peer-urls
      value: "https://${ETCD_IP}:2380"
    - name: initial-advertise-peer-urls
      value: "https://${ETCD_IP}:2380"
    - name: initial-cluster
      value: "${ETCD_INITIAL_CLUSTER}"
    - name: initial-cluster-state
      value: "new"
EOF
```

Check:

```bash
cat /root/kubeadm-etcd.yaml
```

---

### 9.5 Generate Native etcd Certificates on Each etcd Node

Execute on etcd-01, etcd-02, etcd-03:

sudo kubeadm init phase certs etcd-server --config /root/kubeadm-etcd.yaml

    sudo kubeadm init phase certs etcd-peer --config /root/kubeadm-etcd.yaml

    sudo kubeadm init phase certs etcd-healthcheck-client --config /root/kubeadm-etcd.yaml

Check:

    sudo ls -l /etc/kubernetes/pki/etcd/

Should contain at least:

    ca.crt
    ca.key
    server.crt
    server.key
    peer.crt
    peer.key
    healthcheck-client.crt
    healthcheck-client.key

---

### 9.6 Generate apiserver-etcd-client Certificate on etcd-01

Execute on etcd-01:

    sudo kubeadm init phase certs apiserver-etcd-client --config /root/kubeadm-etcd.yaml

Check:

    sudo ls -l /etc/kubernetes/pki/

Should show:

    apiserver-etcd-client.crt
    apiserver-etcd-client.key

---

### 9.7 Generate etcd Static Pod on Each etcd Node

Execute on etcd-01, etcd-02, etcd-03:

    sudo kubeadm init phase etcd local --config /root/kubeadm-etcd.yaml

Check static Pod file:

    sudo ls -l /etc/kubernetes/manifests/

Should show:

    etcd.yaml

Wait for etcd container to start:

    sudo crictl ps | grep etcd

Check logs:

    ETCD_CONTAINER_ID=$(sudo crictl ps --name etcd -q)

    sudo crictl logs ${ETCD_CONTAINER_ID} | tail -n 50

---

### 9.8 Verify External etcd Cluster Health

Execute on etcd-01:

    ETCD_CONTAINER_ID=$(sudo crictl ps --name etcd -q)

    sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
      --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      endpoint health --cluster

Check members:

    sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
      --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      member list

Expect to see 3 etcd members.

Note:

    External etcd nodes will not appear in kubectl get pods -A results.
    Because etcd nodes are not joined to the Kubernetes cluster.
    You must check etcd static Pod using crictl on etcd nodes.

---

### 9.9 Package Certificates for Master to Connect etcd

Execute on etcd-01:

    sudo tar -C /etc/kubernetes/pki -czf /root/master-etcd-client-pki.tar.gz \
      etcd/ca.crt \
      apiserver-etcd-client.crt \
      apiserver-etcd-client.key

Copy certificate package to 3 Master nodes:

    scp /root/master-etcd-client-pki.tar.gz root@k8s-master-01:/root/
    scp /root/master-etcd-client-pki.tar.gz root@k8s-master-02:/root/
    scp /root/master-etcd-client-pki.tar.gz root@k8s-master-03:/root/

Execute on k8s-master-01, k8s-master-02, k8s-master-03:

    sudo mkdir -p /etc/kubernetes/pki
    sudo tar -C /etc/kubernetes/pki -xzf /root/master-etcd-client-pki.tar.gz

Check:

    sudo ls -l /etc/kubernetes/pki/etcd/ca.crt
    sudo ls -l /etc/kubernetes/pki/apiserver-etcd-client.crt
    sudo ls -l /etc/kubernetes/pki/apiserver-etcd-client.key

---

## Ten. Preparing kube-vip for Master-01 Before Initialization

The following operations are executed on k8s-master-01.

Note: /think

kube-vip is not only deployed on master01.
All three Masters must run kube-vip static Pods in the end.
master01 places kube-vip.yaml before kubeadm init to provide the APIServer VIP during initialization.
master02 and master03 place kube-vip.yaml after a successful kubeadm join --control-plane.
Finally, the three kube-vip instances elect one node to hold 10.0.0.30 through leader election.

---

### 10.1 Confirm Network Interface Name

Check the network interfaces:

    ip addr

Confirm the network interface where 10.0.0.20 resides.

Example:

    ens33

If the actual network interface is not ens33, the vip_interface in the kube-vip YAML must be changed to the real interface name.

---

### 10.2 Create kube-vip Static Pod

During the Kubernetes v1.31 initialization phase, it is recommended to first use:

    /etc/kubernetes/super-admin.conf

Switch back to:

    /etc/kubernetes/admin.conf

after kubeadm init completes.

On k8s-master-01, execute:

    sudo mkdir -p /etc/kubernetes/manifests

    cat <<EOF | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      name: kube-vip
      namespace: kube-system
    spec:
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "true"
        - name: port
          value: "6443"
        - name: vip_interface
          value: "ens33"
        - name: vip_cidr
          value: "24"
        - name: cp_enable
          value: "true"
        - name: cp_namespace
          value: "kube-system"
        - name: vip_ddns
          value: "false"
        - name: vip_leaderelection
          value: "true"
        - name: vip_address
          value: "10.0.0.30"
        - name: vip_kubeconfig
          value: "/etc/kubernetes/super-admin.conf"
        image: ghcr.io/kube-vip/kube-vip:v0.8.9
        imagePullPolicy: IfNotPresent
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
        volumeMounts:
        - mountPath: /etc/kubernetes/super-admin.conf
          name: kubeconfig
      hostNetwork: true
      volumes:
      - hostPath:
          path: /etc/kubernetes/super-admin.conf
        name: kubeconfig
    status: {}
    EOF

If the network interface is not ens33, modify:

    sudo sed -i.bak 's/value: "ens33"/value: "实际网卡名"/' /etc/kubernetes/manifests/kube-vip.yaml

Check:

    grep -n "vip_interface" /etc/kubernetes/manifests/kube-vip.yaml
    grep -n "vip_address" /etc/kubernetes/manifests/kube-vip.yaml
    grep -n "super-admin.conf" /etc/kubernetes/manifests/kube-vip.yaml

Note:

    The kube-vip image comes from ghcr.io.
    If the current environment cannot access ghcr.io, it is recommended to synchronize the kube-vip image to an internal Harbor repository in advance and replace the image address in the YAML.
    master01 does not specify type for hostPath before initialization to avoid prematurely creating an empty super-admin.conf.

---

## Eleven. Initialize Kubernetes Cluster on Master-01

The following operations are executed on k8s-master-01.

In external etcd mode, kubeadm init is recommended to use a configuration file because it requires specifying external etcd endpoints and certificate paths.

---

### 11.1 Create kubeadm Configuration File

On k8s-master-01, execute: /think

```bash
cat <<EOF | sudo tee /root/kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "10.0.0.20"
  bindPort: 6443
nodeRegistration:
  name: "k8s-master-01"
  criSocket: "unix:///run/containerd/containerd.sock"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "v1.31.14"
imageRepository: "registry.aliyuncs.com/google_containers"
controlPlaneEndpoint: "k8s-api-server:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
apiServer:
  certSANs:
  - "k8s-api-server"
  - "10.0.0.30"
  - "10.0.0.20"
  - "10.0.0.21"
  - "10.0.0.22"
etcd:
  external:
    endpoints:
    - "https://10.0.0.31:2379"
    - "https://10.0.0.32:2379"
    - "https://10.0.0.33:2379"
    caFile: "/etc/kubernetes/pki/etcd/ca.crt"
    certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
    keyFile: "/etc/kubernetes/pki/apiserver-etcd-client.key"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

**Check:**

```bash
cat /root/kubeadm-config.yaml
```

---

### 11.2 Execute kubeadm init

On k8s-master-01 execute:

```bash
sudo kubeadm init --config /root/kubeadm-config.yaml --upload-certs
```

After initialization completes, save the join command from the output.

---

### 11.3 kube-vip Switch to admin.conf

After successful kubeadm init, switch kube-vip from super-admin.conf to admin.conf:

```bash
sudo cp /etc/kubernetes/manifests/kube-vip.yaml /etc/kubernetes/manifests/kube-vip.yaml.bak.$(date +%F-%H%M%S)

sudo sed -i.bak 's#super-admin.conf#admin.conf#g' /etc/kubernetes/manifests/kube-vip.yaml
```

**Check:**

```bash
grep -n "admin.conf" /etc/kubernetes/manifests/kube-vip.yaml
```

Wait for kubelet to automatically rebuild the kube-vip static Pod:

```bash
sudo crictl ps | grep kube-vip
```

---

### 11.4 Configure kubectl

On k8s-master-01 execute:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Check:**

```bash
kubectl get nodes
```

The node may be NotReady, which is normal since CNI has not been installed yet.

---

### 11.5 View Control Plane Static Pods

Check the manifest file:

```bash
ls -l /etc/kubernetes/manifests/
```

In external etcd mode, the Master node should not generate etcd.yaml.

Expected to see:

```
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
kube-vip.yaml
```

Should not see:

```
etcd.yaml
```

Check the Pod:

```bash
kubectl -n kube-system get pods -o wide
```

Check the containers:

```bash
sudo crictl ps | grep -E "kube-apiserver|kube-controller|kube-scheduler|kube-vip"
```

Check the VIP:

```bash
ip addr | grep 10.0.0.30
```

---

## Twelve. Configure kube-proxy to Use IPVS Mode

The IPVS module was already loaded during the system initialization phase, so here we need to explicitly configure kube-proxy to use IPVS mode.

The following operations are performed on k8s-master-01.

---

### 12.1 Check Current kube-proxy Configuration

Execute:

```bash
kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E "mode:|scheduler:"
```

If you see:

```
mode: ""
```

It indicates that IPVS mode is not explicitly specified.

---

### 12.2 Modify kube-proxy to Use IPVS Mode

Export the kube-proxy ConfigMap:

```bash
kubectl -n kube-system get cm kube-proxy -o yaml > /tmp/kube-proxy.yaml
```

Backup: /think

cp /tmp/kube-proxy.yaml /tmp/kube-proxy.yaml.bak.$(date +%F-%H%M%S)

Modify mode and scheduler:

    sed -i.bak -E \
      -e 's/^([[:space:]]*)mode:.*$/\1mode: "ipvs"/' \
      -e 's/^([[:space:]]*)scheduler:.*$/\1scheduler: "rr"/' \
      /tmp/kube-proxy.yaml

Check:

    grep -E "mode:|scheduler:" /tmp/kube-proxy.yaml

Apply changes:

    kubectl replace -f /tmp/kube-proxy.yaml

---

### 12.3 Restart kube-proxy

Delete kube-proxy Pod, let DaemonSet automatically recreate:

    kubectl -n kube-system delete pod -l k8s-app=kube-proxy

Check:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

---

### 12.4 Verify Configuration

Check configuration:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E "mode:|scheduler:"

Expected output:

    mode: "ipvs"
    scheduler: "rr"

After creating Service, use the following command to verify IPVS rules:

    sudo ipvsadm -Ln

---

## Thirteen, Install Calico Network Plugin

The following operations are performed on k8s-master-01.

The Pod network segment is:

    10.244.0.0/16

The CIDR in Calico configuration must match the podSubnet in kubeadm configuration.

---

### 13.1 Download Calico YAML

Create directory:

    mkdir -p /root/k8s-yaml/calico
    cd /root/k8s-yaml/calico

Download Calico Operator:

    curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

Download custom-resources:

    curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml

If GitHub download is slow, you can download the files in advance on a machine with network access and upload them to this directory.

---

### 13.2 Modify Pod CIDR

Modify custom-resources.yaml:

    sed -i.bak 's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#' custom-resources.yaml

Check:

    grep -n "cidr:" custom-resources.yaml

---

### 13.3 Install Calico

Apply Operator:

    kubectl create -f tigera-operator.yaml

Apply custom resources:

    kubectl create -f custom-resources.yaml

Check status:

    kubectl -n tigera-operator get pods -o wide
    kubectl -n calico-system get pods -o wide

Check nodes:

    kubectl get nodes -o wide

Wait until k8s-master-01 becomes Ready.

Notes:

    If Calico image pull fails, you need to synchronize Calico-related images to internal Harbor in advance, or configure a accessible image acceleration solution.
    This document keeps the main flow clear, image synchronization as a separate production optimization item for later.

---

## Fourteen, Add Master-02 / Master-03 to Control Plane

The following operations are performed separately on k8s-master-02 and k8s-master-03.

Notes:

    Do not place kube-vip.yaml on master02 and master03 before kubeadm join.
    First execute kubeadm join --control-plane.
    After join succeeds, create kube-vip static Pod.
    This ensures /etc/kubernetes/admin.conf already exists.

---

### 14.1 Get Control Plane Join Command

On k8s-master-01, view the join command output from kubeadm init.

If token has expired, you can regenerate it:

    kubeadm token create --print-join-command

If certificate-key has expired, you can re-upload certificates:

    sudo kubeadm init phase upload-certs --upload-certs

Control plane join command format is as follows:

    sudo kubeadm join k8s-api-server:6443 \
      --token <token> \
      --discovery-token-ca-cert-hash sha256:<hash> \
      --control-plane \
      --certificate-key <certificate-key> \
      --apiserver-advertise-address "<current Master real IP>" \
      --cri-socket "unix:///run/containerd/containerd.sock"

Notes:

    kubeadm join does not include --image-repository by default.
    kubeadm init phase has already specified the control plane image repository through kubeadm-config.yaml.

sudo kubeadm join k8s-api-server:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key> \
  --apiserver-advertise-address "10.0.0.21" \
  --cri-socket "unix:///run/containerd/containerd.sock"

Configure kubectl:

    mkdir -p $HOME/.kube

    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

    sudo chown $(id -u):$(id -g) $HOME/.kube/config

---

### 14.3 Create kube-vip Static Pod on k8s-master-02

Execute on k8s-master-02:

    sudo mkdir -p /etc/kubernetes/manifests

    cat <<EOF | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      name: kube-vip
      namespace: kube-system
    spec:
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "true"
        - name: port
          value: "6443"
        - name: vip_interface
          value: "ens33"
        - name: vip_cidr
          value: "24"
        - name: cp_enable
          value: "true"
        - name: cp_namespace
          value: "kube-system"
        - name: vip_ddns
          value: "false"
        - name: vip_leaderelection
          value: "true"
        - name: vip_address
          value: "10.0.0.30"
        - name: vip_kubeconfig
          value: "/etc/kubernetes/admin.conf"
        image: ghcr.io/kube-vip/kube-vip:v0.8.9
        imagePullPolicy: IfNotPresent
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
        volumeMounts:
        - mountPath: /etc/kubernetes/admin.conf
          name: kubeconfig
      hostNetwork: true
      volumes:
      - hostPath:
          path: /etc/kubernetes/admin.conf
          type: File
        name: kubeconfig
    status: {}
    EOF

If the network interface is not ens33, modify:

    sudo sed -i.bak 's/value: "ens33"/value: "actual network interface name"/' /etc/kubernetes/manifests/kube-vip.yaml

Check:

    sudo crictl ps | grep kube-vip
    ip addr | grep 10.0.0.30

---

### 14.4 Join k8s-master-03 to Control Plane

Execute on k8s-master-03:

    sudo kubeadm join k8s-api-server:6443 \
      --token <token> \
      --discovery-token-ca-cert-hash sha256:<hash> \
      --control-plane \
      --certificate-key <certificate-key> \
      --apiserver-advertise-address "10.0.0.22" \
      --cri-socket "unix:///run/containerd/containerd.sock"

Configure kubectl:

    mkdir -p $HOME/.kube

    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

    sudo chown $(id -u):$(id -g) $HOME/.kube/config

---

### 14.5 Create kube-vip Static Pod on k8s-master-03

Execute on k8s-master-03:

    sudo mkdir -p /etc/kubernetes/manifests

```bash
cat <<EOF | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-vip
  namespace: kube-system
spec:
  containers:
  - args:
    - manager
    env:
    - name: vip_arp
      value: "true"
    - name: port
      value: "6443"
    - name: vip_interface
      value: "ens33"
    - name: vip_cidr
      value: "24"
    - name: cp_enable
      value: "true"
    - name: cp_namespace
      value: "kube-system"
    - name: vip_ddns
      value: "false"
    - name: vip_leaderelection
      value: "true"
    - name: vip_address
      value: "10.0.0.30"
    - name: vip_kubeconfig
      value: "/etc/kubernetes/admin.conf"
    image: ghcr.io/kube-vip/kube-vip:v0.8.9
    imagePullPolicy: IfNotPresent
    name: kube-vip
    resources: {}
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW
    volumeMounts:
    - mountPath: /etc/kubernetes/admin.conf
      name: kubeconfig
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/admin.conf
      type: File
    name: kubeconfig
status: {}
EOF
```

If the network interface is not `ens33`, modify:

```bash
sudo sed -i.bak 's/value: "ens33"/value: "actual network interface name"/' /etc/kubernetes/manifests/kube-vip.yaml
```

Check:

```bash
sudo crictl ps | grep kube-vip
ip addr | grep 10.0.0.30
```

---

### 14.6 Check Master Status

Execute on any Master:

```bash
kubectl get nodes -o wide
```

Check control plane Pods:

```bash
kubectl -n kube-system get pods -o wide | grep -E "kube-apiserver|kube-controller|kube-scheduler|kube-vip"
```

In External etcd mode, there should be no etcd Pods in the `kube-system` namespace on Master nodes.

Expected output:

```
3 kube-apiserver
3 kube-controller-manager
3 kube-scheduler
3 kube-vip
```

Check VIP:

```bash
ip addr | grep 10.0.0.30
```

---

## Fifteen、Worker Node Join Cluster

The following operations are performed on `k8s-worker-01`, `k8s-worker-02`, `k8s-worker-03`.

---

### 15.1 Get Worker Join Command

Execute on `k8s-master-01`:

```bash
kubeadm token create --print-join-command
```

Worker join command format:

```
sudo kubeadm join k8s-api-server:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket "unix:///run/containerd/containerd.sock"
```

---

### 15.2 Join Worker Nodes to Cluster

Execute on each Worker node:

```bash
sudo kubeadm join k8s-api-server:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket "unix:///run/containerd/containerd.sock"
```

---

### 15.3 Check Node Status

Execute on Master:

```bash
kubectl get nodes -o wide
```

Expected output:

```
k8s-master-01   Ready   control-plane
k8s-master-02   Ready   control-plane
k8s-master-03   Ready   control-plane
k8s-worker-01   Ready
k8s-worker-02   Ready
k8s-worker-03   Ready
```

Note:

- `etcd-01`, `etcd-02`, `etcd-03` will not appear in `kubectl get nodes`.
- They are independent External etcd nodes and do not join the Kubernetes cluster.

---

## SixteenI don't know.Cluster Basic Validation

---

### 16.1 Check All Pods

Execute:

```bash
kubectl get pods -A -o wide
```

Focus on:

kube-system  
calico-system  
tigera-operator  

Should not exist long-term:  

    Pending  
    CrashLoopBackOff  
    ImagePullBackOff  

In External etcd mode:  

    kubectl get pods -A will not display etcd Pod.  
    etcd Pod must be viewed using crictl on the etcd node.  

---

### 16.2 Verify External etcd  

Execute on etcd-01:  

    sudo crictl ps | grep etcd  

Check etcd health status:  

    ETCD_CONTAINER_ID=$(sudo crictl ps --name etcd -q)  

    sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
      --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      endpoint health --cluster  

Check members:  

    sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
      --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      member list  

Expect to see 3 etcd members.  

---

### 16.3 Verify kube-vip  

Execute on each of the three Masters:  

    ip addr | grep 10.0.0.30  

Normally, only one Master holds the VIP at the same time.  

Access APIServer:  

    curl -k https://k8s-api-server:6443/livez  

Expect to return:  

    ok  

Check version:  

    curl -k https://k8s-api-server:6443/version  

---

### 16.4 Verify kube-proxy IPVS  

Check kube-proxy configuration:  

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E "mode:|scheduler:"  

Expected result:  

    mode: "ipvs"  
    scheduler: "rr"  

Check IPVS rules:  

    sudo ipvsadm -Ln  

If no business Service has been created yet, the rules may be few. After creating Service, more Virtual Server rules should appear.  

---

### 16.5 Verify CoreDNS  

Create a test Pod:  

    kubectl run dns-test --image=busybox:1.36 --restart=Never -- sleep 3600  

Check:  

    kubectl get pod dns-test -o wide  

Test resolution:  

    kubectl exec dns-test -- nslookup kubernetes.default.svc.cluster.local  

Expect to resolve to:  

    10.96.0.1  

Cleanup:  

    kubectl delete pod dns-test  

---

### 16.6 Verify Pod and Service  

Create a test Deployment:  

    kubectl create deployment nginx-test --image=nginx:1.25  

Scale:  

    kubectl scale deployment nginx-test --replicas=3  

Check Pod distribution:  

    kubectl get pods -o wide  

Create Service:  

    kubectl expose deployment nginx-test --port=80 --target-port=80 --type=ClusterIP  

Check Service:  

    kubectl get svc nginx-test  

Create a temporary test Pod:  

    kubectl run curl-test --image=busybox:1.36 --restart=Never -it --rm -- sh  

Execute inside the container:  

    wget -qO- nginx-test.default.svc.cluster.local  

If it returns the nginx page, it indicates:  

    Pod network is normal  
    Service forwarding is normal  
    CoreDNS is normal  
    kube-proxy is normal  

Cleanup:  

    kubectl delete svc nginx-test  
    kubectl delete deployment nginx-test  

---

## Seventeen, kube-vip High Availability Verification  

Note:  

    The following is fault simulation.  
    Production environments must be executed during maintenance windows.  

---

### 17.1 Check current VIP node  

Execute on each of the three Masters:  

    ip addr | grep 10.0.0.30  

Record the Master currently holding the VIP.  

---

### 17.2 Stop kubelet on current VIP node  

Assume the VIP is on k8s-master-01.  

Execute on k8s-master-01:  

    sudo systemctl stop kubelet  

Observe VIP on other Masters:  

    ip addr | grep 10.0.0.30  

Test APIServer:  

    curl -k https://k8s-api-server:6443/livez  

If it returns ok, the VIP has drifted and the control plane entry is still available.  

Restore k8s-master-01:  

    sudo systemctl start kubelet  

Check:  

    kubectl get nodes  
    kubectl get pods -A -o wide  

---

## Eighteen, External etcd Fault Verification  

Note: /think

# Fault Simulation

## 18.1 Stop an etcd Node

For example, execute on etcd-03:

    sudo systemctl stop kubelet

The etcd static Pod on etcd-03 will stop.

Perform health check on etcd-01:

    ETCD_CONTAINER_ID=$(sudo crictl ps --name etcd -q)

    sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
      --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      endpoint health --cluster

If the remaining two etcd members are healthy, the Kubernetes control plane should still be available.

Verify Kubernetes:

    kubectl get nodes
    kubectl get pods -A

Restore etcd-03:

    sudo systemctl start kubelet

Check again:

    sudo crictl ps | grep etcd

---

## Nineteen. Common Troubleshooting

---

### 19.1 External etcd Won't Start

Check on etcd node:

    systemctl status kubelet --no-pager
    journalctl -u kubelet -f
    sudo crictl ps -a
    sudo crictl images

Check manifest:

    sudo cat /etc/kubernetes/manifests/etcd.yaml

Check data directory:

    sudo ls -ld /data/etcd
    sudo du -sh /data/etcd

Common causes:

    1. etcd nodes cannot communicate on port 2380
    2. etcd client cannot communicate on port 2379
    3. certificate SAN does not include current node IP or hostname
    4. initial-cluster configuration error
    5. dataDir permission or path anomaly
    6. containerd not running properly
    7. kubelet static Pod management configuration error

---

### 19.2 kubeadm init Failed

Check on Master node:

    journalctl -u kubelet -f
    systemctl status containerd --no-pager
    sudo crictl info
    sudo crictl images

Check external etcd certificate:

    sudo ls -l /etc/kubernetes/pki/etcd/ca.crt
    sudo ls -l /etc/kubernetes/pki/apiserver-etcd-client.crt
    sudo ls -l /etc/kubernetes/pki/apiserver-etcd-client.key

Check etcd connectivity:

    telnet 10.0.0.31 2379
    telnet 10.0.0.32 2379
    telnet 10.0.0.33 2379

Common causes:

    1. Master lacks external etcd client certificate
    2. etcd endpoints in kubeadm-config.yaml are incorrect
    3. etcd port 2379 unreachable
    4. etcd cluster itself is unhealthy
    5. imageRepository not properly configured
    6. kube-vip network interface name is incorrect

---

### 19.3 kube-vip Not Taking Effect

Check kube-vip manifest:

    cat /etc/kubernetes/manifests/kube-vip.yaml

Check network interface:

    ip addr

Check kube-vip container:

    sudo crictl ps | grep kube-vip
    sudo crictl ps -a | grep kube-vip

View logs:

    sudo crictl logs <kube-vip-container-id>

Check VIP:

    ip addr | grep 10.0.0.30

Common causes:

    1. vip_interface is incorrect
    2. 10.0.0.30 is already occupied
    3. kube-vip image pull failed
    4. current network does not support ARP VIP drift
    5. super-admin.conf not used before master01 initialization
    6. kube-vip not switched back to admin.conf after kubeadm init
    7. kube-vip static Pod not deployed after master02/master03 join

---

### 19.4 Master Join Failed

Check VIP reachability:

    ping -c 3 k8s-api-server
    curl -k https://k8s-api-server:6443/livez

Check token:

    kubeadm token list

Regenerate token:

    kubeadm token create --print-join-command

Re-upload certificate:

    sudo kubeadm init phase upload-certs --upload-certs

Common causes:

    1. token expired
    2. certificate-key expired
    3. VIP unreachable
    4. k8s-api-server resolution error
    5. time synchronization issue
    6. kube-vip.yaml incorrectly placed before master02/master03 join

---

### 19.5 Service Unreachable

Check Service:

    kubectl get svc
    kubectl describe svc <svc-name>

Check Endpoints:

kubectl get endpoints <svc-name>

Check Pod labels:

    kubectl get pods --show-labels

Check IPVS:

    sudo ipvsadm -Ln

Common causes:

    1. Service selector is written incorrectly
    2. Pod label mismatch
    3. Endpoints are empty
    4. targetPort is written incorrectly
    5. The service inside the container is not listening on the corresponding port
    6. kube-proxy anomaly

---

### 19.6 containerd Data Directory Not Taking Effect

Check configuration:

    grep -n '^root = ' /etc/containerd/config.toml

Expected result:

    root = "/data/containerd"

Check directory:

    sudo ls -ld /data/containerd
    sudo du -sh /data/containerd

Check containerd status:

    systemctl status containerd --no-pager

Common causes:

    1. Containerd was not restarted after configuration changes
    2. sed did not match the root configuration
    3. /data directory was not created
    4. /data is not mounted independently and still occupies root partition space
    5. The directory was modified after the cluster was already running, with old data still in /var/lib/containerd

---

## Twenty, Deployment Completion Checklist

After deployment, check each item sequentially:

    kubectl get nodes -o wide
    kubectl get pods -A -o wide
    kubectl -n kube-system get pods -o wide | grep kube-vip
    kubectl -n calico-system get pods -o wide
    curl -k https://k8s-api-server:6443/livez
    sudo ipvsadm -Ln
    grep -n '^root = ' /etc/containerd/config.toml
    sudo du -sh /data/containerd

Check on etcd-01:

    sudo crictl ps | grep etcd

    ETCD_CONTAINER_ID=$(sudo crictl ps --name etcd -q)

    sudo crictl exec ${ETCD_CONTAINER_ID} etcdctl \
      --endpoints=https://10.0.0.31:2379,https://10.0.0.32:2379,https://10.0.0.33:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      endpoint health --cluster

Should satisfy:

    1. 3 Master nodes Ready
    2. Worker nodes Ready
    3. 3 kube-vip static Pods are running normally
    4. 10.0.0.30 exists only on one Master
    5. Master nodes do not run local etcd Pods
    6. 3 External etcd nodes are healthy
    7. Calico is running normally
    8. CoreDNS resolves normally
    9. kube-proxy uses IPVS mode
    10. Pods can communicate across nodes
    11. Services can be accessed normally
    12. containerd data directory is /data/containerd
    13. etcd data directory is /data/etcd

---

## Twenty-one, Recommendations for Installing Subsequent Components

This article only completes the deployment of the Kubernetes high-availability cluster itself.

Recommended subsequent installations:

    1. metrics-server
    2. ingress-nginx
    3. cert-manager
    4. StorageClass
    5. Prometheus / Grafana
    6. Loki / ELK
    7. Gateway API Controller

Recommended next article:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/01-ingress-nginx Production Installation: NodePort, IngressClass, and Access Verification.md

---

## Twenty-two, Summary

This article completes a deployment process for a kubeadm External etcd high-availability Kubernetes cluster suitable for the domestic environment.

Core features:

    1. Ubuntu 22.04
    2. kubeadm deployment
    3. 3 Master high-availability control plane
    4. 3 External etcd independent nodes
    5. etcd uses kubeadm-generated static Pod
    6. etcd is managed by the local kubelet on etcd nodes
    7. kube-vip provides APIServer VIP
    8. kube-vip runs on all 3 Master nodes
    9. kube-vip uses super-admin.conf before master01 initialization
    10. After kubeadm init, kube-vip switches to admin.conf
    11. kube-vip static Pod is deployed after master02/master03 join
    12. containerd as the container runtime
    13. containerd data directory adjusted to /data/containerd
    14. etcd data directory adjusted to /data/etcd
    15. Alibaba Cloud Docker CE source installs containerd
    16. Alibaba Cloud Kubernetes source installs kubeadm/kubelet/kubectl
    17. Alibaba Cloud google_containers image repository initializes the cluster
    18. kube-proxy fixed to IPVS mode
    19. Calico provides Pod network
    20. Use sed -i.bak uniformly when modifying configurations for easy rollback

This note serves as a foundation Runbook for subsequent re-deployments of Kubernetes External etcd high-availability clusters.