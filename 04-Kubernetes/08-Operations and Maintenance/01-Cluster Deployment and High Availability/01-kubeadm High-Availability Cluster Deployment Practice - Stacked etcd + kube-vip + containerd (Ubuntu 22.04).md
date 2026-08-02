# 01-kubeadm High-Availability Cluster Deployment Practice: Stacked etcd + kube-vip + containerd (Ubuntu 22.04)

Recommended Path:

    04-Kubernetes/08-Operations/01-Cluster Deployment and High Availability/01-kubeadm High-Availability Cluster Deployment Practice: Stacked etcd + kube-vip + containerd (Ubuntu 22.04).md

Tags:

    #Kubernetes
    #kubeadm
    #High-Availability Cluster
    #Stacked-etcd
    #kube-vip
    #containerd
    #IPVS
    #Ubuntu2204
    #Domestic Environment
    #Cluster Deployment

---

## I. Document Purpose

This document records a process for deploying a highly available Kubernetes cluster using kubeadm.

This solution is designed for domestic environments and utilizes:

    Ubuntu Server 22.04
    containerd
    kubeadm / kubelet / kubectl
    Alibaba Cloud Docker CE repository
    Alibaba Cloud Kubernetes repository
    Alibaba Cloud google_containers image repository
    kube-vip for providing APIServer VIP
    Stacked etcd high-availability mode
    kube-proxy in IPVS mode
    containerd data directory set to /data/containerd

The goal of this document is to provide a step-by-step guide for deploying highly available Kubernetes clusters that can be easily followed in practice.

---

## II. Node Planning

### 2.1 Cluster Planning

| Item | Planning |
|---|---|
| Operating System | Ubuntu Server 22.04 |
| Kubernetes Version | v1.31.14 |
| Container Runtime | containerd |
| Deployment Tool | kubeadm |
| Control Plane | 3 Masters |
| etcd Mode | Stacked etcd |
| APIServer High Availability | kube-vip |
| APIServer VIP | 10.0.0.30 |
| APIServer Domain Name | k8s-api-server |
| Pod IP Range | 10.244.0.0/16 |
| Service IP Range | 10.96.0.0/12 |
| kube-proxy Mode | IPVS |
| CNI Plugin | Calico |
| containerd Data Directory | /data/containerd |

---

### 2.2 Node IP Planning

| Role | Hostname | IP Address | Notes |
|---|---|---|---|
| Operations Tool Node | ops-server | 10.0.0.10 | Optional for GitLab, Jenkins, Harbor |
| Master 1 | k8s-master-01 | 10.0.0.20 | Control-plane + etcd |
| Master 2 | k8s-master-02 | 10.0.0.21 | Control-plane + etcd |
| Master 3 | k8s-master-03 | 10.0.0.22 | Control-plane + etcd |
| Worker 1 | k8s-worker-01 | 10.0.0.23 | Work node |
| Worker 2 | k8s-worker-02 | 10.0.0.24 | Work node |
| Worker 3 | k8s-worker-03 | 10.0.0.25 | Work node, optional |
| APIServer VIP | k8s-api-server | 10.0.0.30 | Floating address for kube-vip |

Notes:

    10.0.0.30 is not assigned to any real server; it is only used as a virtual IP for kube-vip.
    Both `kubeadm init` and `kubeadm join` access the control plane via `k8s-api-server:6443`.

---

### 2.3 Architecture Diagram

```txt
    kubectl / kubelet / Operations Tool
                  |
                  |
          k8s-api-server:6443
                  |
                  |
             VIP: 10.0.0.30
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
    kube-controller-manager     kube```bash
sudo hostnamectl set-hostname k8s-worker-01

Execute on k8s-worker-02:

sudo hostnamectl set-hostname k8s-worker-02

Execute on k8s-worker-03:

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

## Section 5: System Initialization for All Nodes

The following steps are to be performed on all Master and Worker nodes.

---

### 5.1 Configure Clock Synchronization

Install chrony:

sudo apt update
sudo apt install -y chrony

Replace with the domestic NTP source:

sudo sed -i.bak '/^pool ntp\.ubuntu\.com\|^pool [012]\.ubuntu\.pool\.ntp\.org/s/^/#/;/^#pool ntp\.ubuntu\.com/i\pool ntp.aliyun.com iburst\npool ntp.huaweicloud.com iburst\npool ntp.tencent.com iburst\npool time.windows.com iburst' /etc/chrony/chrony.conf

Start and set to start at boot:

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

Immediately load the modules:

sudo modprobe overlay
sudo modprobe br_netfilter
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe nf_conntrack

Check:

lsmod | grep -E "overlay|br_netfilter|ip_vs|nf_conntrack"

Explanation:

overlay and br_netfilter are essential for Kubernetes network functionality via containerd.
ip_vs, ip_vs_rr, ip_vs_wrr, ip_vs_sh, and nf_conntrack are required for the kube-proxy in IPVS mode.
Later in this document, kube-proxy will be explicitly configured to use IPVS mode.

---

### 5.4 Configure Kernel Parameters

Write sysctl settings:

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
netbridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

Load the configuration:

sudo sysctl --system

Check:

sysctl net_bridge.bridge-nf-call-iptables
sysctl net_bridge.bridge-nf-call-ip6tables
net.ipv4.ip_forward

Expected results:

net.bridge.bridge-nf-call-iptables = 1
netbridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1

---

### 5.5 Install Basic Dependencies

Install commonly used dependencies for Kubernetes nodes:

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

Explanation:

conntrack is a connection tracking tool required by kube-proxy.
socat is often used for functions like kubectl exec and kubectl port-forward.
ipset and ipvsadm are useful for troubleshooting with kube-proxy in IPVS mode.
nfs-common is needed for futureThe text you've provided appears to be a set of instructions for setting up and configuring a Kubernetes environment using various tools and software. I will translate the content as requested, preserving the original formatting and technical details.

### 6.5 Enabling SystemdCgroup

Modify the configuration:

```bash
sudo sed -i.bak 's/SystemdCgroup = false(SystemdCgroup = true)/g' /etc/containerd/config.toml
```

Check:

```bash
grep "SystemdCgroup" /etc/containerd/config.toml
```

Expected result:

```
SystemdCgroup = true
```

### 6.6 Replacing Pause Images with Alibaba Cloud Sources

Replace the `pause` images in the containerd configuration with Alibaba Cloud images:

```bash
sudo sed -i.bak -E 's#registry.k8s.io/pause:[0-9.]+#registry.aliyuncs.com/google_containers/pause:3.10#g' /etc/containerd/config.toml
```

Verify the Sandbox image configuration:

```bash
containerd config dump | grep -E "sandbox_image|pinned_helper_image"
```

If `registry.k8s.io/pause` is still seen, manually check:

```bash
grep -n "pause" /etc/containerd/config.toml
```

### 6.7 Restarting Containerd

Restart containerd:

```bash
sudo systemctl restart containerd
```

Set it to start automatically at boot:

```bash
sudo systemctl enable containerd
```

Check the status:

```bash
systemctl status containerd --no-pager
```

Check the data directory:

```bash
sudo ls -ld /data/containerd
```

### 6.8 Configuring Crichtl

Write the Crichtl configuration:

```bash
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

Check:

```bash
sudo crictl info
```

View images:

```bash
sudo crictl images
```

## VII. Installing Nerdctl on All Nodes

Nerdctl is a Docker-style command-line tool for containerd, useful for managing container images, containers, and namespaces.

Perform the following steps on all Master and Worker nodes:

### 7.1 Installing the Nerdctl Minimal Package

Set the version:

```bash
NERDCTL_VERSION=2.2.2
```

Download:

```bash
cd /tmp
wget https://github.com/containerd/nerdctl/releases/download/v${NERDCTL_VERSION}/nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz
```

Install:

```bash
sudo tar -C /usr/local/bin -xzf nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz
```

Check:

```bash
nerdctl --version
```

If the GitHub download is slow, you can download the `nerdctl` tar package in advance and manually transfer it to the nodes before extracting it.

### 7.2 Common Check Commands

View default namespace images:

```bash
sudo nerdctl images
```

View Kubernetes `k8s.io` namespace images:

```bash
sudo nerdctl -n k8s.io images
```

View Kubernetes containers:

```bash
sudo nerdctl -n k8s.io ps
```

View all Kubernetes containers:

```bash
sudo nerdctl -n k8s.io ps -a
```

Note:

Kubernetes-related containers are usually in the `k8s.io` namespace. When troubleshooting Kubernetes images and containers, it is recommended to use `sudo nerdctl -n k8s.io`.

## VIII. Installing Kubeadm, Kubelet, and Kubectl on All Nodes

Perform the following steps on all Master and Worker nodes.

This document uses the Alibaba Cloud Kubernetes v1.31 software source.

### 8.1 Configuring the Alibaba Cloud Kubernetes Source

Install basic dependencies:

```bash
sudo apt-get update
sudo apt-get install -y \
      apt-transport-https \
      ca-certificates \
      curl \
      gpg
```

Create a keyring directory:

```bash
sudo mkdir -p /etc/apt/keyrings
```

Add the Kubernetes GPG key:

```bash
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.31/deb/Release.key | \
      sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes v1.31 apt source:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://kube-vip is not only deployed on master01. Ultimately, all three Masters need to have the kube-vip static Pod running. On master01, the kube-vip.yaml file is placed before executing kubeadm init to provide the APIServer VIP during initialization. For master02 and master03, the kube-vip.yaml file is added after successfully executing kubeadm join --control-plane. In the end, through leader election, one of the three nodes will be assigned the IP address 10.0.0.30.

---

### 9.1 Confirming the Network Card Name

To view the network cards:

    ip addr

Confirm the name of the network card that holds the IP address 10.0.0.20.

Example:

    ens33

If the actual network card is not ens33, the vip_interface field in the kube-vip.yaml file must be updated to reflect the correct network card name.

---

### 9.2 Creating the kube-vip Static Pod

During the initialization phase of Kubernetes v1.31, it is recommended to use the following configuration initially:

    /etc/kubernetes/super-admin.conf

After kubeadm init is complete, switch back to using:

    /etc/kubernetes/admin.conf

On k8s-master-01, execute the following commands:

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

If the network card is not ens33, modify the file as follows:

    sudo sed -i.bak 's/value: "ens33"/value: "actual network card name"/' /etc/kubernetes/manifests/kube-vip.yaml

Verify the following settings:

    grep -n "vip_interface" /etc/kubernetes/manifests/kube-vip.yaml
    grep -n "vip_address" /etc/kubernetes/manifests/kube-vip.yaml
    grep -n "super-admin.conf" /etc/kubernetes/manifests/kube-vip.yaml

Note:

    The kube-vip image is available from ghcr.io. If your environment cannot access this repository, it is advisable to download the image to a local Harbor and update the corresponding path in the YAML file.

    On master01, do not specify the type of the hostPath volume before initialization to prevent the creation of an empty super-admin.conf file prematurely.

---

## Section X: Master-01 Initializing the Cluster

The following steps should be performed on k8s-master-01.

---

### 10.1 Executing kubeadm init

Use the following command to initialize the cluster:

    sudo kubeadm init \
      --control-plane-endpoint "k8s-api-server:6443" \
      --apiserver-advertise-address "10.0.0.20" \
      --apiserver-cert-extra-sans "k8s-api-server,10.0.0.30,10.0.0.20,10.0.0.21,10.0.0.22" \
      --image-repository "registry.aliyuncs.com/google_containers" \
      --kubernetes-version "v1.31.14"```markdown
ls -l /etc/kubernetes/manifests/

Expected output:

etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
kube-vip.yaml

View Pods:

kubectl -n kube-system get pods -o wide

View Containers:

sudo crictl ps | grep -E "kube-apiserver|etcd|kube-controller|kube-scheduler|kube-vip"

View VIP:

ip addr | grep 10.0.0.30

---

## Chapter Eleven: Configuring kube-proxy to Use IPVS Mode

The IPVS module was already loaded during the system initialization phase, so it is necessary to explicitly configure kube-proxy to use IPVS mode.

The following steps are performed on k8s-master-01.

---

### 11.1 Checking the Current kube-proxy Configuration

Execute:

kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep -E "mode:|scheduler:"

If you see:

mode: ""

it means that the IPVS mode is not explicitly specified currently.

---

### 11.2 Changing kube-proxy to Use IPVS Mode

Export the kube-proxy ConfigMap:

kubectl -n kube-system get cm kube-proxy -o yaml > /tmp/kube-proxy.yaml

Back up the file:

cp /tmp/kube-proxy.yaml /tmp/kube-proxy.yaml.bak.$(date +%F-%H%M%S)

Modify the "mode" and "scheduler" fields:

sed -i.bak -E \
  -e 's/^([[:space:]]*)mode:.*$/\1mode: "ipvs"/' \
  -e 's/^([[:space:]]*)scheduler:.*$/\1scheduler: "rr"/' \
  /tmp/kube-proxy.yaml

Check the changes:

grep -E "mode:|scheduler:" /tmp/kube-proxy.yaml

Apply the modifications:

kubectl replace -f /tmp/kube-proxy.yaml

---

### 11.3 Restarting kube-proxy

Delete the kube-proxy Pod to allow the DaemonSet to rebuild automatically:

kubectl -n kube-system delete pod -l k8s-app=kube-proxy

View the status:

kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

---

### 11.4 Verifying the Configuration

Check the configuration again:

kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep -E "mode:|scheduler:"

Expected output:

mode: "ipvs"
scheduler: "rr"

After creating a Service later on, you can use the following command to verify the IPVS rules:

sudo ipvsadm -Ln

---

## Chapter Twelve: Installing the Calico Network Plugin

The following steps are performed on k8s-master-01.

The Pod network segment in this document is:

10.244.0.0/16

The CIDR range specified in the Calico configuration must match the --pod-network-cidr parameter used during kubeadm init.

---

### 12.1 Downloading Calico YAML Files

Create a directory:

mkdir -p /root/k8s-yaml/calico
cd /root/k8s-yaml/calico

Download the Calico Operator:

curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

Download the custom-resources file:

curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml

If the download from GitHub is slow, you can download the files in advance on a machine with an internet connection and then copy them to this directory.

---

### 12.2 Modifying the Pod CIDR Range

Edit the custom-resources.yaml file:

sed -i.bak 's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#' custom-resources.yaml

Check the changes:

grep -n "cidr:" custom-resources.yaml

---

### 12.3 Installing Calico

Apply the Operator:

kubectl create -f tigera-operator.yaml

Apply the custom resources:

kubectl create -f custom-resources.yaml

Check the status:

kubectl -n tigera-operator get pods -o wide
kubectl -n calico-system get pods -o wide

View the nodes:

kubectl get nodes -o wide

Wait for k8s-master-01 to become Ready.

Note:

If there is an issue with downloading the Calico images, you need to synchronize the relevant images to the internal Harbor in advance, or configure an accessible image acceleration solution.
For clarity in this document, the image synchronization process is discussed separately as a potential optimization for production environments.

---

## Chapter Thirteen```bash
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

If the network card is not "ens33", modify the following line:

    sudo sed -i.bak 's/value: "ens33"/value: "actual network card name"/' /etc/kubernetes/manifests/kube-vip.yaml

Verify the configuration by executing the following commands:

    sudo crictl ps | grep kube-vip
    ip addr | grep 10.0.0.30
```The command format for adding a Worker node is as follows:

    sudo kubeadm join k8s-api-server:6443 \
      --token <token> \
      --discovery-token-ca-cert-hash sha256:<hash> \
      --cri-socket "unix:///run/containerd/containerd.sock"

Note:

Worker nodes do not run kube-apiserver, controller-manager, scheduler, or etcd. To keep the kubeadm join command concise, the --image-repository option is not included here. The image addresses for components such as kube-proxy are determined by the cluster configuration during the kubeadm init phase.

---

### 14.2 Adding Worker Nodes to the Cluster

Execute the following command on each Worker node:

    sudo kubeadm join k8s-api-server:6443 \
      --token <token> \
      --discovery-token-ca-cert-hash sha256:<hash> \
      --cri-socket "unix:///run/containerd/containerd.sock"

---

### 14.3 Checking Node Status

On the Master nodes, execute the following command:

    kubectl get nodes -o wide

You should see the following output:

    k8s-master-01   Ready   control-plane
    k8s-master-02   Ready   control-plane
    k8s-master-03   Ready   control-plane
    k8s-worker-01   Ready
    k8s-worker-02   Ready
    k8s-worker-03   Ready

---

## Section 15: Basic Cluster Verification

---

### 15.1 Viewing All Pods

Execute the following command:

    kubectl get pods -A -o wide

Pay special attention to the following pods:

    kube-system
    calico-system
    tigera-operator

These pods should not have any status such as Pending, CrashLoopBackOff, or ImagePullBackOff for an extended period.

---

### 15.2 Verifying etcd

View the etcd Pod:

    kubectl -n kube-system get pods -o wide | grep etcd

Check the health status of etcd:

    kubectl -n kube-system exec etcd-k8s-master-01 -- \
      etcdctl \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
      endpoint health --cluster

View the etcd members:

    kubectl -n kube-system exec etcd-k8s-master-01 -- \
      etcdctl \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
      member list

You should see 3 etcd members listed.

---

### 15.3 Verifying kube-vip

Execute the following command on each Master node:

    ip addr | grep 10.0.0.30

Under normal circumstances, only one Master node should have the VIP address at any given time.

Access the APIServer:

    curl -k https://k8s-api-server:6443/livez

You should receive a response of "ok".

Check the version:

    curl -k https://k8s-api-server:6443/version

---

### 15.4 Verifying kube-proxy IPVS

View the kube-proxy configuration:

    kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep -E "mode:|scheduler:"

The expected result is:

    mode: "ipvs"
    scheduler: "rr"

View the IPVS rules:

    sudo ipvsadm -Ln

If no business Services have been created yet, there may be few rules listed. More Virtual Server rules should appear after creating Services.

---

### 15.5 Verifying CoreDNS

Create a test Pod:

    kubectl run dns-test --image=busybox:1.36 --restart=Never -- sleep 3600

View the Pod:

    kubectl get pod dns-test -o wide

Test DNS resolution:

    kubectl exec dns-test -- nslookup kubernetes.default.svc.cluster.local

The expected response is:

    10.96.0.1

Clean up:

    kubectl delete pod dns-test

---

### 15.6 Verifying Pods and Services

Create a test Deployment:

    kubectl create deployment nginx-test --image=nginx:1.25

Scale the Deployment:

   Check Port Occupancy:

    sudo ss -lntp | grep -E "6443|2379|2380|10250"

Common Causes:

    1. Image cannot be pulled
    2. Incorrect containerd configuration
    3. Abnormal pause of images
    4. Incorrect hostname or hosts configuration
    5. Port is already in use
    6. Wrong kube-vip network interface name
    7. Incorrect kubeconfig path during kube-vip initialization

---

### 17.3 kube-vip Not Functioning

Check the kube-vip manifest:

    cat /etc/kubernetes/manifests/kube-vip.yaml

Check the network interfaces:

    ip addr

Check kube-vip containers:

    sudo crictl ps | grep kube-vip
    sudo crictl ps -a | grep kube-vip

View logs:

    sudo crictl logs <kube-vip-container-id>

Check the VIP:

    ip addr | grep 10.0.0.30

Common Causes:

    1. Incorrect vip_interface specified
    2. 10.0.0.30 is already in use
    3. Failed to pull the kube-vip image
    4. Current network does not support ARP VIP migration
    5. master01 did not use super-admin.conf before initialization
    6.kube-vip was not switched back to admin.conf after kubeadm init
    7. kube-vip static Pod was not deployed after master02 / master03 joined the cluster

---

### 17.4 Master Nodes Cannot Join the Cluster

Check if the VIP is reachable:

    ping -c 3 k8s-api-server
    curl -k https://k8s-api-server:6443/livez

Check the token:

    kubeadm token list

Regenerate the token:

    kubeadm token create --print-join-command

Reupload certificates:

    sudo kubeadm init phase upload-certs --upload-certs

Common Causes:

    1. Token has expired
    2. Certificate-key has expired
    3. VIP is unreachable
    4. k8s-api-server parsing error
    5. Time synchronization issues
    6. kube-vip.yaml was incorrectly placed before master02 / master03 joined the cluster

---

### 17.5 Services Are Not Accessible

Check Services:

    kubectl get svc
    kubectl describe svc <svc-name>

Check Endpoints:

    kubectl get endpoints <svc-name>

Check Pod labels:

    kubectl get pods --show-labels

Check IPVS:

    sudo ipvsadm -Ln

Common Causes:

    1. Incorrect Service selector specified
    2. Pod labels do not match
    3. Endpoints are empty
    4. TargetPort is incorrectly specified
    5. Services inside containers are not listening on the correct ports
    6. kube-proxy is malfunctioning

---

### 17.6 containerd Data Directory Is Not Effective

Check the configuration:

    grep -n '^root = ' /etc/containerd/config.toml

Expected result:

    root = "/data/containerd"

Check the directory:

    sudo ls -ld /data/containerd
    sudo du -sh /data/containerd

Check containerd status:

    systemctl status containerd --no-pager

Common Causes:

    1. Containerd was not restarted after configuration changes
    2. The sed command did not match the root configuration setting
    3. /data directory was not created
    4. /data is not mounted separately and still occupies space in the root partition
    5. Changes were made after the cluster was already running, so old data remains in /var/lib/containerd

---

## Section Eighteen: Post-Deployment Inspection Checklist

After deployment, perform the following checks:

    kubectl get nodes -o wide
    kubectl get pods -A -o wide
    kubectl -n kube-system get pods -o wide | grep etcd
    kubectl -n kube-system get pods -o wide | grep kube-vip
    kubectl -n calico-system get pods -o wide
    curl -k https://k8s-api-server:6443/livez
    sudo ipvsadm -Ln
    grep -n '^root = ' /etc/containerd/config.toml
    sudo du -sh /data/containerd

Expected results:

    1. All 3 Master nodes are in the Ready state
    2. Worker nodes are also in the Ready state
    3. All 3 kube-vip static Pods are running normally
    4. 10.0.0