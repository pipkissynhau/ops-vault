# 01-kubeadm High Availability Cluster Deployment: Stacked etcd + kube-vip + containerd (Ubuntu 22.04)

Recommended path:

    04-Kubernetes/08-Operations/01-Cluster Deployment and High Availability/01-kubeadm High Availability Cluster Deployment: Stacked etcd + kube-vip + containerd (Ubuntu 22.04).md

Tags:

    #Kubernetes
    #kubeadm
    #HighAvailableClusters
    #Stacked-etcd
    #kube-vip
    #containerd
    #IPVS
    #Ubuntu2204
    #DomesticEnvironment
    #ClusterDeployment

---

## I. Document Objectives

This document records the deployment process for a Kubernetes high availability cluster based on kubeadm.

This solution is tailored for domestic environments, using:

    Ubuntu Server 22.04
    containerd
    kubeadm / kubelet / kubectl
    Alibaba Cloud Docker CE source
    Alibaba Cloud Kubernetes source
    Alibaba Cloud google_containers image repository
    kube-vip for APIServer VIP
    Stacked etcd high availability mode
    kube-proxy IPVS mode
    containerd data directory adjusted to /data/containerd

The objective of this document is to create a deployment note that can be followed step-by-step for a Kubernetes high availability cluster.

---

## II. Node Planning

### 2.1 Cluster Planning

| Item | Plan |
|---|---|
| Operating System | Ubuntu Server 22.04 |
| Kubernetes Version | v1.31.14 |
| Container Runtime | containerd |
| Deployment Tool | kubeadm |
| Control Plane | 3 Master |
| etcd Mode | Stacked etcd |
| APIServer High Availability | kube-vip |
| APIServer VIP | 10.0.0.30 |
| APIServer Domain | k8s-api-server |
| Pod Network Segment | 10.244.0.0/16 |
| Service Network Segment | 10.96.0.0/12 |
| kube-proxy Mode | IPVS |
| CNI Plugin | Calico |
| containerd Data Directory | /data/containerd |

---

### 2.2 Node IP Planning

| Role | Hostname | IP Address | Notes |
|---|---|---:|---|
| Operations Tool Node | ops-server | 10.0.0.10 | GitLab / Jenkins / Harbor, optional |
| Master 1 | k8s-master-01 | 10.0.0.20 | control-plane + etcd |
| Master 2 | k8s-master-02 | 10.0.0.21 | control-plane + etcd |
| Master 3 | k8s-master-03 | 10.0.0.22 | control-plane + etcd |
| Worker 1 | k8s-worker-01 | 10.0.0.23 | Worker node |
| Worker 2 | k8s-worker-02 | 10.0.0.24 | Worker node |
| Worker 3 | k8s-worker-03 | 10.0.0.25 | Worker node, optional |
| APIServer VIP | k8s-api-server | 10.0.0.30 | kube-vip floating IP |

Notes:

    10.0.0.30 is not assigned to any real server.
    10.0.0.30 is only used as a virtual IP for kube-vip.
    kubeadm init and kubeadm join access the control plane through k8s-api-server:6443.

---

### 2.3 Architecture Diagram

```txt
    kubectl / kubelet / Operational tools
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
    etcd                        etcd           etcd
    kube-vip                    kube-vip       kube-vip

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
```

---

## III. Pre-Deployment Checks

Confirm on all nodes:

    ip addr
    hostname
    free -h
    df -h
    timedatectl
    cat /etc/os-release

Verification items:

    1. All node IPs are fixed
    2. All node hostnames are unique
    3. All nodes have synchronized time
    4. All nodes can communicate with each other
    5. All nodes can access domestic software sources
    6. 10.0.0.30 is not occupied by any host
    7. /data is recommended to be mounted as an independent data disk or large capacity partition

Check node communication:

    ping -c 3 10.0.0.20
    ping -c 3 10.0.0.21
    ping -c 3 10.0.0.22
    ping -c 3 10.0.0.23
    ping -c 3 10.0.0.24
    ping -c 3 10.0.0.25

Check /data:

    sudo mkdir -p /data
    df -h /data

Notes:

    If /data is not mounted independently, but just a regular directory under the root partition, containerd data will still occupy system disk space.
    In production environments, it is recommended to mount /data to an independent data disk.

---

## IV. Configure Hostnames and hosts on All Nodes

The following operations are performed on all Master and Worker nodes.

Node range:

    k8s-master-01
    k8s-master-02
    k8s-master-03
    k8s-worker-01
    k8s-worker-02
    k8s-worker-03

---

### 4.1 Set Hostnames

On k8s-master-01:

    sudo hostnamectl set-hostname k8s-master-01

On k8s-master-02:

    sudo hostnamectl set-hostname k8s-master-02

On k8s-master-03:

    sudo hostnamectl set-hostname k8s-master-03

On k8s-worker-01:

    sudo hostnamectl set-hostname k8s-worker-01

On k8s-worker-02:

    sudo hostnamectl set-hostname k8s-worker-02

On k8s-worker-03:

sudo hostnamectl set-hostname k8s-worker-03

Check:

    hostname

---

### 4.2 Configure /etc/hosts

All nodes execute:

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
    EOF

Check resolution:

    getent hosts k8s-master-01
    getent hosts k8s-master-02
    getent hosts k8s-master-03
    getent hosts k8s-api-server

---

## Five. System Initialization for All Nodes

The following operations are executed on all Master and Worker nodes.

---

### 5.1 Configure Clock Synchronization

Install chrony:

    sudo apt update
    sudo apt install -y chrony

Replace with domestic NTP source:

    sudo sed -i.bak '/^pool ntp\.ubuntu\.com\|^pool [012]\.ubuntu\.pool\.ntp\.org/s/^/#/;/^#pool ntp\.ubuntu\.com/i\pool ntp.aliyun.com iburst\npool ntp.huaweicloud.com iburst\npool ntp.tencent.com iburst\npool time.windows.com iburst' /etc/chrony/chrony.conf

Start and set to enable on boot:

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

If swapon --show has no output, Swap is disabled.

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

### 5.5 Install Base Dependencies

Install common dependencies for Kubernetes nodes:

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

    conntrack is a connection tracking tool required by kube-proxy.
    socat is commonly used for kubectl exec, kubectl port-forward, etc.
    ipset and ipvsadm are used for troubleshooting kube-proxy IPVS mode.
    nfs-common is used for subsequent NFS storage integration.

---

## Six. Install containerd on All Nodes

The following operations are executed on all Master and Worker nodes.

This document uses Alibaba Cloud Docker CE source to install containerd.io.

Note:

    Kubernetes nodes use containerd as the container runtime.
    This document only installs containerd.io, not the full Docker Engine.
    containerd data directory is uniformly adjusted to /data/containerd.

---

### 6.1 Configure Alibaba Cloud Docker CE Source

Create keyrings directory:

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

By default, containerd's data directory is:

    /var/lib/containerd

This directory stores:

    1. Image layer data
    2. Container snapshot data
    3. containerd metadata
    4. Kubernetes Pod image cache

It is not recommended to use the system root partition for containerd image data in production environments. It is recommended to adjust to:

    /data/containerd

Create directory:

    sudo mkdir -p /data/containerd

Modify containerd root path:

    sudo sed -i.bak 's#^root = "/var/lib/containerd"#root = "/data/containerd"#g' /etc/containerd/config.toml

Check:

    grep -n '^root = ' /etc/containerd/config.toml

Expected result:

    root = "/data/containerd"

Note:

    root is containerd's persistent data directory.
    Image, snapshot, metadata, etc., main data will be stored in this directory.
    state defaults to /run/containerd, which is a runtime state directory and generally does not need modification.
    It is best to complete this configuration before kubeadm init.
    If the cluster is already running, migrating containerd data directory requires stopping kubelet and containerd, and carefully handling existing container data.

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

Write crictl configuration:

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

## VII. Install nerdctl on All Nodes

nerdctl is a Docker-style CLI tool for containerd, suitable for troubleshooting containerd images, containers, and namespaces.

The following operations are performed on all Master and Worker nodes.

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

If downloading from GitHub is slow, you can download the nerdctl tar package in advance and manually upload it to the nodes for extraction.

---

### 7.2 Common inspection commands

View default namespace images:

    sudo nerdctl images

View Kubernetes-used k8s.io namespace images:

    sudo nerdctl -n k8s.io images

View Kubernetes containers:

    sudo nerdctl -n k8s.io ps

View all Kubernetes containers:

    sudo nerdctl -n k8s.io ps -a

Note:

    Kubernetes-related containers are typically in the k8s.io namespace.
    When troubleshooting Kubernetes images and containers, it is recommended to use sudo nerdctl -n k8s.io.

---

## VIII. Install kubeadm, kubelet, kubectl on all nodes

The following operations are performed on all Master and Worker nodes.

This document uses the Aliyun Kubernetes v1.31 software repository.

---

### 8.1 Configure Aliyun Kubernetes source

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

If apt-cache madison shows 1.31.14-1.1, it is recommended to specify installing this version.

---

### 8.3 Install kubeadm, kubelet, kubectl

Install with specified version:

    sudo apt-get install -y \
      kubelet=1.31.14-1.1 \
      kubeadm=1.31.14-1.1 \
      kubectl=1.31.14-1.1

If the software source does not contain 1.31.14-1.1, you can use the latest patch version from the current v1.31 source:

    sudo apt-get install -y kubelet kubeadm kubectl

Check versions:

    kubeadm version
    kubelet --version
    kubectl version --client

Note:

    The --kubernetes-version in kubeadm init must match the actual installed version.
    If the installed version is not v1.31.14, you need to synchronize and modify the version number in the kubeadm init command.

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

    At this point, kubelet may not be in the Running state, which is normal.
    kubelet requires kubeadm init or kubeadm join to obtain complete configuration.

---

## IX. Preparing kube-vip on Master-01 before initialization

The following operations are performed on k8s-master-01.

Note:

    kube-vip is not deployed only on master01.
    All three Masters ultimately need to run kube-vip static Pods.
    master01 places kube-vip.yaml before kubeadm init to provide the APIServer VIP during initialization.
    master02 and master03 place kube-vip.yaml after kubeadm join --control-plane succeeds.
    The three kube-vip instances will elect one node to hold 10.0.0.30 through leader election.

---

### 9.1 Confirm network interface name

Check network interfaces:

    ip addr

Confirm the network interface where 10.0.0.20 resides.

Example:

    ens33

If the actual network interface is not ens33, the vip_interface in the kube-vip YAML must be changed to the real network interface name.

---

### 9.2 Create kube-vip static Pod

For Kubernetes v1.31 initialization phase, it is recommended to first use:

    /etc/kubernetes/super-admin.conf

Switch back to:

    /etc/kubernetes/admin.conf

after kubeadm init completes.

On k8s-master-01 execute: /think

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

sudo sed -i.bak 's/value: "ens33"/value: "actual network interface name"/' /etc/kubernetes/manifests/kube-vip.yaml

Check:

grep -n "vip_interface" /etc/kubernetes/manifests/kube-vip.yaml
grep -n "vip_address" /etc/kubernetes/manifests/kube-vip.yaml
grep -n "super-admin.conf" /etc/kubernetes/manifests/kube-vip.yaml

Note:

kube-vip image comes from ghcr.io.
If the current environment cannot access ghcr.io, it is recommended to synchronize the kube-vip image to an internal Harbor in advance and replace the image address in the YAML.
Before master01 initialization, hostPath does not specify type to avoid creating an empty super-admin.conf file.

---

## Ten. Master-01 Initialize Cluster

The following operations are performed on k8s-master-01.

---

### 10.1 Execute kubeadm init

Initialization command:

sudo kubeadm init \
  --control-plane-endpoint "k8s-api-server:6443" \
  --apiserver-advertise-address "10.0.0.20" \
  --apiserver-cert-extra-sans "k8s-api-server,10.0.0.30,10.0.0.20,10.0.0.21,10.0.0.22" \
  --image-repository "registry.aliyuncs.com/google_containers" \
  --kubernetes-version "v1.31.14" \
  --pod-network-cidr "10.244.0.0/16" \
  --service-cidr "10.96.0.0/12" \
  --cri-socket "unix:///run/containerd/containerd.sock" \
  --upload-certs

Parameter explanation:

--control-plane-endpoint
    High availability control plane entry point, must specify the VIP domain.

--apiserver-advertise-address
    Real IP of the current Master.

--apiserver-cert-extra-sans
    Additional SANs for APIServer certificate, recommend including VIP, domain name, and all Master IPs.

--image-repository
    Specify the domestic control plane image repository.

--kubernetes-version
    Specify Kubernetes version.

--pod-network-cidr
    Specify Pod network segment, needs to be consistent with CNI configuration.

--service-cidr
    Specify Service network segment.

--cri-socket
    Specify containerd CRI socket.

--upload-certs
    Upload control plane certificates, convenient for other Masters to join.

After initialization, save the join command output.

### 10.2 Switch kube-vip Back to admin.conf

After kubeadm init is successful, switch kube-vip from super-admin.conf back to admin.conf:

    sudo cp /etc/kubernetes/manifests/kube-vip.yaml /etc/kubernetes/manifests/kube-vip.yaml.bak.$(date +%F-%H%M%S)

    sudo sed -i.bak 's#super-admin.conf#admin.conf#g' /etc/kubernetes/manifests/kube-vip.yaml

Check:

    grep -n "admin.conf" /etc/kubernetes/manifests/kube-vip.yaml

Wait for kubelet to automatically rebuild the kube-vip static Pod:

    sudo crictl ps | grep kube-vip

---

### 10.3 Configure kubectl

Execute on k8s-master-01:

    mkdir -p $HOME/.kube

    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

    sudo chown $(id -u):$(id -g) $HOME/.kube/config

Check:

    kubectl get nodes

The nodes may be NotReady, which is normal because CNI has not been installed yet.

---

### 10.4 View Static Pods

View the manifest file:

    ls -l /etc/kubernetes/manifests/

Expected to see:

    etcd.yaml
    kube-apiserver.yaml
    kube-controller-manager.yaml
    kube-scheduler.yaml
    kube-vip.yaml

View Pods:

    kubectl -n kube-system get pods -o wide

View containers:

    sudo crictl ps | grep -E "kube-apiserver|etcd|kube-controller|kube-scheduler|kube-vip"

View VIP:

    ip addr | grep 10.0.0.30

---

## Eleven, Configure kube-proxy to Use IPVS Mode

The IPVS module was already loaded during the system initialization phase, so here we need to explicitly configure kube-proxy to use IPVS mode.

The following operations are executed on k8s-master-01.

---

### 11.1 Check Current kube-proxy Configuration

Execute:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config.conf}' | grep -E "mode:|scheduler:"

If you see:

    mode: ""

It indicates that IPVS mode is not explicitly specified.

---

### 11.2 Modify kube-proxy to Use IPVS Mode

Export kube-proxy ConfigMap:

    kubectl -n kube-system get cm kube-proxy -o yaml > /tmp/kube-proxy.yaml

Backup:

    cp /tmp/kube-proxy.yaml /tmp/kube-proxy.yaml.bak.$(date +%F-%H%M%S)

Modify mode and scheduler:

    sed -i.bak -E \
      -e 's/^([[:space:]]*)mode:.*$/\1mode: "ipvs"/' \
      -e 's/^([[:space:]]*)scheduler:.*$/\1scheduler: "rr"/' \
      /tmp/kube-proxy.yaml

Check:

    grep -E "mode:|scheduler:" /tmp/kube-proxy.yaml

Apply the changes:

    kubectl replace -f /tmp/kube-proxy.yaml

---

### 11.3 Restart kube-proxy

Delete the kube-proxy Pod to let DaemonSet automatically rebuild it:

    kubectl -n kube-system delete pod -l k8s-app=kube-proxy

Check:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

---

### 11.4 Verify the Configuration

Check the configuration:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config.conf}' | grep -E "mode:|scheduler:"

Expected to see:

    mode: "ipvs"
    scheduler: "rr"

After creating a Service later, you can use the following command to verify IPVS rules:

    sudo ipvsadm -Ln

---

## Twelve, Install Calico Network Plugin

The following operations are executed on k8s-master-01.

The Pod network segment in this document is:

    10.244.0.0/16

The CIDR in Calico configuration must be consistent with the --pod-network-cidr specified during kubeadm init.

---

### 12.1 Download Calico YAML

Create a directory:

    mkdir -p /root/k8s-yaml/calico
    cd /root/k8s-yaml/calico

Download Calico Operator:

    curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

Download custom-resources:

    curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml

If downloading from GitHub is slow, you can download the files in advance on a machine with network access and upload them to this directory.

---

### 12.2 Modify Pod CIDR

Modify custom-resources.yaml:

    sed -i.bak 's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#' custom-resources.yaml

Check:

    grep -n "cidr:" custom-resources.yaml

---

### 12.3 Install Calico

Apply Operator: /think

kubectl create -f tigera-operator.yaml

Apply Custom Resources:

    kubectl create -f custom-resources.yaml

Check Status:

    kubectl -n tigera-operator get pods -o wide
    kubectl -n calico-system get pods -o wide

Check Nodes:

    kubectl get nodes -o wide

Wait until k8s-master-01 becomes Ready.

Note:

    If Calico image pull fails, you need to pre-synchronize Calico-related images to internal Harbor or configure a accessible image acceleration solution.
    This article maintains the main flow clarity, and image synchronization is treated as a separate production optimization item for later.

---

## ThirteenI don't know.Master-02 / Master-03 Join Control Plane

The following operations are executed separately on k8s-master-02 and k8s-master-03.

Note:

    Do not place kube-vip.yaml on master02 and master03 before kubeadm join.
    First execute kubeadm join --control-plane.
    Create kube-vip static Pod after join succeeds.
    This ensures /etc/kubernetes/admin.conf already exists.

---

### 13.1 Get Control Plane Join Command

On k8s-master-01, view the join command output from kubeadm init.

If token expires, you can regenerate it:

    kubeadm token create --print-join-command

If certificate-key expires, you can re-upload certificates:

    sudo kubeadm init phase upload-certs --upload-certs

Control plane join command format is as follows:

    sudo kubeadm join k8s-api-server:6443 \
      --token <token> \
      --discovery-token-ca-cert-hash sha256:<hash> \
      --control-plane \
      --certificate-key <certificate-key> \
      --apiserver-advertise-address "<Current Master Real IP>" \
      --cri-socket "unix:///run/containerd/containerd.sock"

Note:

    kubeadm join defaults not to include --image-repository.
    kubeadm init phase has already specified the control plane image repository via --image-repository.
    If current environment's kubeadm join --help explicitly supports --image-repository, you can append this parameter according to actual version.

---

### 13.2 k8s-master-02 Join Control Plane

Execute on k8s-master-02:

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

### 13.3 Create kube-vip Static Pod on k8s-master-02

Execute on k8s-master-02:

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

### 13.4 Adding k8s-master-03 to the Control Plane

On `k8s-master-03`, execute:

```bash
sudo kubeadm join k8s-api-server:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key> \
  --apiserver-advertise-address "10.0.0.22" \
  --cri-socket "unix:///run/containerd/containerd.sock"
```

Configure `kubectl`:

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 13.5 Creating kube-vip Static Pod on k8s-master-03

On `k8s-master-03`, execute:

```bash
sudo mkdir -p /etc/kubernetes/manifests
```

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

### 13.6 Check Master Status

Execute on any Master:

```bash
kubectl get nodes -o wide
```

Check control plane Pods:

```bash
kubectl -n kube-system get pods -o wide | grep -E "kube-apiserver|kube-controller|kube-scheduler|etcd|kube-vip"
```

Expected output:

```
3 kube-apiserver
3 kube-controller-manager
3 kube-scheduler
3 etcd
3 kube-vip
```

Check VIP:

```bash
ip addr | grep 10.0.0.30
```

**Note:**  
Only one Master node should hold `10.0.0.30` at the same time.  
If none of the Masters have the VIP, focus on checking kube-vip logs, network interface names, and image pull status.

---

## Fourteen、Worker Node Join Cluster

The following operations are executed on `k8s-worker-01`, `k8s-worker-02`, `k8s-worker-03`.

---

### 14.1 Get Worker Join Command

Execute on `k8s-master-01`:

```bash
kubeadm token create --print-join-command
```

Worker join command format:

```bash
sudo kubeadm join k8s-api-server:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket "unix:///run/containerd/containerd.sock"
```

**Note:**  
Worker nodes do not run `kube-apiserver`, `controller-manager`, `scheduler`, or `etcd`.  
This keeps the `kubeadm join` command concise without extra `--image-repository`.  
kube-proxy and other component image repositories are determined by the cluster configuration during `kubeadm init`.

---

### 14.2 Join Worker Nodes to Cluster

Execute on each Worker node:

```bash
sudo kubeadm join k8s-api-server:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket "unix:///run/containerd/containerd.sock"
```

---

### 14.3 Check Node Status

Execute on Master:

```bash
kubectl get nodes -o wide
```

**Expected Output:**

k8s-master-01   Ready   control-plane
k8s-master-02   Ready   control-plane
k8s-master-03   Ready   control-plane
k8s-worker-01   Ready
k8s-worker-02   Ready
k8s-worker-03   Ready

---

## Fifteen, Cluster Basic Verification

---

### 15.1 Check All Pods

Execute:

    kubectl get pods -A -o wide

Focus on checking:

    kube-system
    calico-system
    tigera-operator

Should not have long-term existence:

    Pending
    CrashLoopBackOff
    ImagePullBackOff

---

### 15.2 Verify etcd

Check etcd Pod:

    kubectl -n kube-system get pods -o wide | grep etcd

Check etcd health status:

    kubectl -n kube-system exec etcd-k8s-master-01 -- \
      etcdctl \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      endpoint health --cluster

Check etcd members:

    kubectl -n kube-system exec etcd-k8s-master-01 -- \
      etcdctl \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      member list

Expect to see 3 etcd members.

---

### 15.3 Verify kube-vip

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

### 15.4 Verify kube-proxy IPVS

Check kube-proxy configuration:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E "mode:|scheduler:"

Expected result:

    mode: "ipvs"
    scheduler: "rr"

Check IPVS rules:

    sudo ipvsadm -Ln

If no business Service has been created yet, the rules may be few. After creating Service, more Virtual Server rules should appear.

---

### 15.5 Verify CoreDNS

Create test Pod:

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

### 15.6 Verify Pod and Service

Create test Deployment:

    kubectl create deployment nginx-test --image=nginx:1.25

Scale up:

    kubectl scale deployment nginx-test --replicas=3

Check Pod distribution:

    kubectl get pods -o wide

Create Service:

    kubectl expose deployment nginx-test --port=80 --target-port=80 --type=ClusterIP

Check Service:

    kubectl get svc nginx-test

Create temporary test Pod:

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

## Sixteen, kube-vip High Availability Verification

Note:

    The following is fault simulation.
    Must be executed in maintenance window in production environment.

---

### 16.1 Check Current VIP Node

Execute on each of the three Masters:

    ip addr | grep 10.0.0.30

Record the Master currently holding the VIP.

---

### 16.2 Stop kubelet on Current VIP Node

Assume the VIP is on k8s-master-01.

Execute on k8s-master-01:

    sudo systemctl stop kubelet

Observe VIP on other Masters:

    ip addr | grep 10.0.0.30

Test APIServer: /think

curl -k https://k8s-api-server:6443/livez

If the response is ok, it indicates that the VIP has drifted and the control plane entry point is still available.

Restore k8s-master-01:

    sudo systemctl start kubelet

Check:

    kubectl get nodes
    kubectl get pods -A -o wide

---

## Seventeen. Common Issues Troubleshooting

---

### 17.1 Node NotReady

Check:

    kubectl describe node <node-name>
    journalctl -u kubelet -f
    sudo crictl ps -a
    sudo crictl images

Common causes:

    1. Swap not closed
    2. containerd not started
    3. CNI not installed
    4. Calico Pod abnormal
    5. Pod CIDR and Calico CIDR inconsistent
    6. kubelet and containerd cgroup driver inconsistent

---

### 17.2 kubeadm init Failed

Check:

    journalctl -u kubelet -f
    systemctl status containerd --no-pager
    sudo crictl info
    sudo crictl images

Check port occupation:

    sudo ss -lntp | grep -E "6443|2379|2380|10250"

Common causes:

    1. Image cannot be pulled
    2. containerd configuration error
    3. pause image abnormal
    4. Hostname or hosts configuration error
    5. Port occupied
    6. kube-vip network interface name written incorrectly
    7. kube-vip initialization phase kubeconfig path incorrect

---

### 17.3 kube-vip Not Effective

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

    1. vip_interface written incorrectly
    2. 10.0.0.30 already occupied
    3. kube-vip image pull failed
    4. Current network does not support ARP VIP drift
    5. master01 did not use super-admin.conf before initialization
    6. After kubeadm init, kube-vip was not switched back to admin.conf
    7. master02/master03 join did not deploy kube-vip static Pod

---

### 17.4 Master Join Failed

Check if VIP is reachable:

    ping -c 3 k8s-api-server
    curl -k https://k8s-api-server:6443/livez

Check token:

    kubeadm token list

Regenerate token:

    kubeadm token create --print-join-command

Re-upload certificate:

    sudo kubeadm init phase upload-certs --upload-certs

Common causes:

    1. Token expired
    2. certificate-key expired
    3. VIP unreachable
    4. k8s-api-server resolution error
    5. Time synchronization error
    6. kube-vip.yaml incorrectly placed before master02/master03 join

---

### 17.5 Service Unreachable

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

    1. Service selector written incorrectly
    2. Pod label mismatch
    3. Endpoints empty
    4. targetPort written incorrectly
    5. Container service not listening on corresponding port
    6. kube-proxy abnormal

---

### 17.6 containerd Data Directory Not Effective

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

    1. Containerd not restarted after configuration change
    2. sed did not match root configuration
    3. /data directory not created
    4. /data not independently mounted, still occupying root partition space
    5. Changed directory after cluster deployment, old data still in /var/lib/containerd

---

## Eighteen. Deployment Completion Checklist

After deployment, check each item sequentially: /think

kubectl get nodes -o wide  
kubectl get pods -A -o wide  
kubectl -n kube-system get pods -o wide | grep etcd  
kubectl -n kube-system get pods -o wide | grep kube-vip  
kubectl -n calico-system get pods -o wide  
curl -k https://k8s-api-server:6443/livez  
sudo ipvsadm -Ln  
grep -n '^root = ' /etc/containerd/config.toml  
sudo du -sh /data/containerd  

---

## 19. Recommendations for Installing Subsequent Components  

This document only completes the deployment of the Kubernetes high-availability cluster itself.  

Recommended subsequent installations:  

    1. metrics-server  
    2. ingress-nginx  
    3. cert-manager  
    4. StorageClass  
    5. Prometheus / Grafana  
    6. Loki / ELK  
    7. Gateway API Controller  

Suggested next article:  

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/01-ingress-nginx Production Installation: NodePort, IngressClass, and Access Verification.md  

---

## 20. Summary  

This document completes a deployment process for a high-availability Kubernetes cluster using kubeadm, tailored for domestic environments.  

Core features:  

    1. Ubuntu 22.04  
    2. kubeadm deployment  
    3. 3 Master high-availability control plane  
    4. Stacked etcd  
    5. kube-vip provides APIServer VIP  
    6. kube-vip runs on all 3 Master nodes finally  
    7. kube-vip uses super-admin.conf before master01 initialization  
    8. kube-vip switches to admin.conf after kubeadm init  
    9. kube-vip static Pod is deployed after master02/master03 join  
    10. containerd as container runtime  
    11. containerd data directory adjusted to /data/containerd  
    12. Alibaba Cloud Docker CE source installs containerd  
    13. Alibaba Cloud Kubernetes source installs kubeadm/kubelet/kubectl  
    14. Alibaba Cloud google_containers image repository initializes the cluster  
    15. kube-proxy fixed to IPVS mode  
    16. Calico provides Pod network  
    17. Use sed -i.bak uniformly for configuration changes, facilitating rollback  

This note serves as a foundation Runbook for subsequent Kubernetes high-availability cluster deployments.